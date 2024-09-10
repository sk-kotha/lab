Certainly! I'll modify the `processChunk` method to return a `List<Record>` containing two records: one with the record ID and a sum of 1 and market value 0, and the other with the actual sum and market value. Here's the updated implementation:

### Updated FileParsingService

```java
import org.springframework.stereotype.Service;
import java.io.IOException;
import java.nio.file.Files;
import java.nio.file.Path;
import java.util.ArrayList;
import java.util.List;
import java.util.stream.Stream;

@Service
public class FileParsingService {

    private final RecordRepository recordRepository;
    private final FileConfigService fileConfigService;

    public FileParsingService(RecordRepository recordRepository, FileConfigService fileConfigService) {
        this.recordRepository = recordRepository;
        this.fileConfigService = fileConfigService;
    }

    public void parseAndSaveFile(String filePath, String fileType) throws IOException {
        FileConfig config = fileConfigService.getConfig(fileType);
        List<Record> records = new ArrayList<>();
        try (Stream<String> lines = Files.lines(Path.of(filePath))) {
            List<String> chunk = new ArrayList<>(config.getChunkSize());
            long[] lineCount = {0};

            lines.forEach(line -> {
                lineCount[0]++;
                chunk.add(line);
                if (chunk.size() == config.getChunkSize()) {
                    records.addAll(processChunk(chunk, config));
                    chunk.clear();
                }
            });

            if (!chunk.isEmpty()) {
                throw new IllegalStateException("File does not contain a multiple of " + config.getChunkSize() + " lines. Total lines: " + lineCount[0]);
            }
        }
        recordRepository.saveAll(records);
    }

    private List<Record> processChunk(List<String> chunk, FileConfig config) {
        if (chunk.size() != config.getChunkSize()) {
            throw new IllegalStateException("Chunk size is not " + config.getChunkSize() + ". Found: " + chunk.size());
        }

        String recordId = chunk.get(0).substring(config.getRecordIdStart(), config.getRecordIdEnd()).trim();
        
        for (int i = 1; i < config.getChunkSize(); i++) {
            if (!chunk.get(i).substring(config.getRecordIdStart(), config.getRecordIdEnd()).trim().equals(recordId)) {
                throw new IllegalStateException("Inconsistent recordId in chunk starting with: " + recordId);
            }
        }

        double marketValue = Double.parseDouble(chunk.get(config.getMarketValueLineIndex())
                .substring(config.getMarketValueStart(), config.getMarketValueEnd()).trim());

        String sumLine = chunk.get(config.getSumLineIndex());
        double sum = calculateSum(sumLine, config.getSumParts());

        List<Record> records = new ArrayList<>();
        records.add(new Record(recordId, 0.0, 1.0)); // Record with sum 1 and market value 0
        records.add(new Record(recordId, marketValue, sum)); // Record with actual sum and market value

        return records;
    }

    private double calculateSum(String row, List<FileConfig.SumPart> sumParts) {
        return sumParts.stream()
                .mapToDouble(part -> Double.parseDouble(row.substring(part.getStart(), part.getEnd()).trim()))
                .sum();
    }
}
```

### Updated Tests

```java
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.mockito.Mock;
import org.mockito.MockitoAnnotations;

import java.io.IOException;
import java.nio.file.Files;
import java.nio.file.Path;
import java.util.List;
import java.util.stream.Stream;

import static org.junit.jupiter.api.Assertions.*;
import static org.mockito.Mockito.*;

class FileParsingServiceTest {

    @Mock
    private RecordRepository recordRepository;

    @Mock
    private FileConfigService fileConfigService;

    private FileParsingService fileParsingService;

    @BeforeEach
    void setUp() {
        MockitoAnnotations.openMocks(this);
        fileParsingService = new FileParsingService(recordRepository, fileConfigService);

        when(fileConfigService.getConfig("TYPE1")).thenReturn(new FileConfig("TYPE1", 4, 0, 30, 49, 70, 0,
            List.of(
                new FileConfig.SumPart(29, 40),
                new FileConfig.SumPart(40, 50),
                new FileConfig.SumPart(50, 60)
            ), 3
        ));
        when(fileConfigService.getConfig("TYPE2")).thenReturn(new FileConfig("TYPE2", 5, 0, 25, 40, 60, 1,
            List.of(
                new FileConfig.SumPart(25, 35),
                new FileConfig.SumPart(35, 45),
                new FileConfig.SumPart(45, 55),
                new FileConfig.SumPart(55, 65)
            ), 4
        ));
    }

    @Test
    void testParseAndSaveFileType1() throws IOException {
        String filePath = "path/to/your/huge/file1.txt";
        
        Stream<String> lines = Stream.of(
                "recordId1                            some data here 12345678901234567890",
                "recordId1                            some data here",
                "recordId1                            some data here",
                "recordId1                            1234567890 1234567890 1234567890"
        );
        try (var mocked = mockStatic(Files.class)) {
            mocked.when(() -> Files.lines(Path.of(filePath))).thenReturn(lines);

            fileParsingService.parseAndSaveFile(filePath, "TYPE1");

            verify(recordRepository, times(1)).saveAll(argThat(records -> {
                assertEquals(2, records.size());
                Record record1 = records.get(0);
                assertEquals("recordId1", record1.recordId());
                assertEquals(0.0, record1.marketValue());
                assertEquals(1.0, record1.sum());

                Record record2 = records.get(1);
                assertEquals("recordId1", record2.recordId());
                assertEquals(12345678901234567890.0, record2.marketValue());
                assertEquals(3703703700.0, record2.sum());
                return true;
            }));
        }
    }

    @Test
    void testParseAndSaveFileType2() throws IOException {
        String filePath = "path/to/your/huge/file2.txt";
        
        Stream<String> lines = Stream.of(
                "recordId2               some data",
                "recordId2               9876543210987654",
                "recordId2               some data",
                "recordId2               some data",
                "recordId2               1234567890 1234567890 1234567890 1234567890"
        );
        try (var mocked = mockStatic(Files.class)) {
            mocked.when(() -> Files.lines(Path.of(filePath))).thenReturn(lines);

            fileParsingService.parseAndSaveFile(filePath, "TYPE2");

            verify(recordRepository, times(1)).saveAll(argThat(records -> {
                assertEquals(2, records.size());
                Record record1 = records.get(0);
                assertEquals("recordId2", record1.recordId());
                assertEquals(0.0, record1.marketValue());
                assertEquals(1.0, record1.sum());

                Record record2 = records.get(1);
                assertEquals("recordId2", record2.recordId());
                assertEquals(9876543210987654.0, record2.marketValue());
                assertEquals(4938271600.0, record2.sum());  // Sum of 4 parts
                return true;
            }));
        }
    }

    @Test
    void testIncompleteChunkThrowsException() throws IOException {
        String filePath = "path/to/your/huge/file1.txt";
        
        Stream<String> lines = Stream.of(
                "recordId1                            some data here 12345678901234567890",
                "recordId1                            some data here",
                "recordId1                            some data here"
        );
        try (var mocked = mockStatic(Files.class)) {
            mocked.when(() -> Files.lines(Path.of(filePath))).thenReturn(lines);

            IllegalStateException exception = assertThrows(IllegalStateException.class, () -> {
                fileParsingService.parseAndSaveFile(filePath, "TYPE1");
            });

            assertEquals("File does not contain a multiple of 4 lines. Total lines: 3", exception.getMessage());
            verify(recordRepository, never()).saveAll(any());
        }
    }

    @Test
    void testInconsistentRecordIdThrowsException() throws IOException {
        String filePath = "path/to/your/huge/file1.txt";
        
        Stream<String> lines = Stream.of(
                "recordId1                            some data here 12345678901234567890",
                "recordId1                            some data here",
                "recordId1                            some data here",
                "recordId2                            1234567890 1234567890 1234567890"
        );
        try (var mocked = mockStatic(Files.class)) {
            mocked.when(() -> Files.lines(Path.of(filePath))).thenReturn(lines);

            IllegalStateException exception = assertThrows(IllegalStateException.class, () -> {
                fileParsingService.parseAndSaveFile(filePath, "TYPE1");
            });

            assertEquals("Inconsistent recordId in chunk starting with: recordId1", exception.getMessage());
            verify(recordRepository, never()).saveAll(any());
        }
    }

    @Test
    void testInvalidMarketValueThrowsException() throws IOException {
        String filePath = "path/to/your/huge/file1.txt";
        
        Stream<String> lines = Stream.of(
                "recordId1                            some data here invalid_value",
                "recordId1                            some data here",
                "recordId1                            some data here",
                "recordId1                            1234567890 1234567890 1234567890"
        );
        try (var mocked = mockStatic(Files.class)) {
            mocked.when(() -> Files.lines(Path.of(filePath))).thenReturn(lines);

            NumberFormatException exception = assertThrows(NumberFormatException.class, () -> {
                fileParsingService.parseAndSaveFile(filePath, "TYPE1");
            });

            assertTrue(exception.getMessage().contains("For input string: \"invalid_value\""));
            verify(recordRepository, never()).saveAll(any());
        }
    }

    @Test
    void testInvalidSumPartThrowsException() throws IOException {
        String filePath = "path/to/your/huge/file1.txt";
        
        Stream<String> lines = Stream.of(
                "recordId1                            some data here 12345678901234567890",
                "recordId1                            some data here",
                "recordId1                            some data here",
                "recordId1                            invalid_value 1234567890 1234567890"
        );
        try (var mocked = mockStatic(Files.class)) {
            mocked.when(() -> Files.lines(Path.of(filePath))).thenReturn(lines);

            NumberFormatException exception = assertThrows(NumberFormatException.class, () -> {
                fileParsingService.parseAndSaveFile(filePath, "TYPE1");
            });

            assertTrue(exception.getMessage().contains("For input string: \"invalid_value\""));
            verify(recordRepository, never()).saveAll(any());
        }
    }

    @Test
    void testUnsupportedFileTypeThrowsException() {
        String filePath = "path/to/your/huge/file3.txt";
        
        IllegalArgumentException exception = assertThrows(IllegalArgumentException.class, () -> {
            fileParsingService.parseAndSaveFile(filePath, "UNSUPPORTED_TYPE");
        });

        assertEquals("Unsupported file type: UNSUPPORTED_TYPE", exception.getMessage());
    }

    // Add more tests for edge cases as needed
}
```

### Summary of Changes

1. **processChunk Method**: Now returns a `List<Record>` containing two records: one with the record ID and a sum of 1 and market value 0, and the other with the actual sum and market value.
2. **Tests**: Updated to verify that two records are created for each chunk and to cover various error cases and edge cases.

These changes ensure that the `processChunk` method creates the required records and that the tests comprehensively cover different scenarios, including error cases and edge cases.
