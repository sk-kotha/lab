
### Constants

Define the constants in a separate class or within the `FileParsingService` class:

```java
public class FileParsingConstants {
    public static final int RECORD_ID_START = 0;
    public static final int RECORD_ID_END = 30;
    public static final int MARKET_VALUE_START = 49;
    public static final int MARKET_VALUE_END = 70;
    public static final int SUM_PART1_START = 29;
    public static final int SUM_PART1_END = 40;
    public static final int SUM_PART2_START = 40;
    public static final int SUM_PART2_END = 50;
    public static final int SUM_PART3_START = 50;
    public static final int SUM_PART3_END = 60;
}
```

### Service Class

Update the `FileParsingService` class to use these constants:

```java
import java.io.IOException;
import java.nio.file.Files;
import java.nio.file.Path;
import java.util.ArrayList;
import java.util.List;
import java.util.stream.Stream;

public class FileParsingService {

    private final RecordRepository recordRepository;

    public FileParsingService(RecordRepository recordRepository) {
        this.recordRepository = recordRepository;
    }

    public void parseAndSaveFile(String filePath) throws IOException {
        List<Record> records = new ArrayList<>();
        try (Stream<String> lines = Files.lines(Path.of(filePath))) {
            List<String> chunk = new ArrayList<>(4);
            long[] lineCount = {0};

            lines.forEach(line -> {
                lineCount[0]++;
                chunk.add(line);
                if (chunk.size() == 4) {
                    records.add(processChunk(chunk));
                    chunk.clear();
                }
            });

            if (!chunk.isEmpty()) {
                throw new IllegalStateException("File does not contain a multiple of 4 lines. Total lines: " + lineCount[0]);
            }
        }
        recordRepository.saveAll(records);
    }

    private Record processChunk(List<String> chunk) {
        if (chunk.size() != 4) {
            throw new IllegalStateException("Chunk size is not 4. Found: " + chunk.size());
        }

        String recordId = chunk.get(0).substring(FileParsingConstants.RECORD_ID_START, FileParsingConstants.RECORD_ID_END).trim();
        
        for (int i = 1; i < 4; i++) {
            if (!chunk.get(i).substring(FileParsingConstants.RECORD_ID_START, FileParsingConstants.RECORD_ID_END).trim().equals(recordId)) {
                throw new IllegalStateException("Inconsistent recordId in chunk starting with: " + recordId);
            }
        }

        double marketValue = Double.parseDouble(chunk.get(0).substring(FileParsingConstants.MARKET_VALUE_START, FileParsingConstants.MARKET_VALUE_END).trim());

        String fourthRow = chunk.get(3);
        double sum = Double.parseDouble(fourthRow.substring(FileParsingConstants.SUM_PART1_START, FileParsingConstants.SUM_PART1_END).trim()) +
                     Double.parseDouble(fourthRow.substring(FileParsingConstants.SUM_PART2_START, FileParsingConstants.SUM_PART2_END).trim()) +
                     Double.parseDouble(fourthRow.substring(FileParsingConstants.SUM_PART3_START, FileParsingConstants.SUM_PART3_END).trim());

        return new Record(recordId, marketValue, sum);
    }
}
```

### Repository Class

The `RecordRepository` class remains unchanged:

```java
import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.stereotype.Repository;
import org.springframework.transaction.annotation.Transactional;

import java.util.List;

@Repository
public class RecordRepository {

    private final JdbcTemplate jdbcTemplate;

    public RecordRepository(JdbcTemplate jdbcTemplate) {
        this.jdbcTemplate = jdbcTemplate;
    }

    @Transactional
    public void saveAll(List<Record> records) {
        String sql = "INSERT INTO records (record_id, market_value, sum) VALUES (?, ?, ?)";
        jdbcTemplate.batchUpdate(sql, records, records.size(),
            (ps, record) -> {
                ps.setString(1, record.recordId());
                ps.setDouble(2, record.marketValue());
                ps.setDouble(3, record.sum());
            });
    }
}
```

### Main Class

The `HugeFileParser` class remains unchanged:

```java
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.context.ApplicationContext;

@SpringBootApplication
public class HugeFileParser {

    public static void main(String[] args) {
        ApplicationContext context = SpringApplication.run(HugeFileParser.class, args);
        FileParsingService fileParsingService = context.getBean(FileParsingService.class);

        String filePath = "path/to/your/huge/file.txt";
        try {
            fileParsingService.parseAndSaveFile(filePath);
            System.out.println("File parsed and records saved successfully.");
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

### Updated Tests

The tests remain largely unchanged, but for completeness, here they are again:

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

    private FileParsingService fileParsingService;

    @BeforeEach
    void setUp() {
        MockitoAnnotations.openMocks(this);
        fileParsingService = new FileParsingService(recordRepository);
    }

    @Test
    void testParseAndSaveFile() throws IOException {
        String filePath = "path/to/your/huge/file.txt";
        
        Stream<String> lines = Stream.of(
                "recordId1                            some data here 12345678901234567890",
                "recordId1                            some data here",
                "recordId1                            some data here",
                "recordId1                            1234567890 1234567890 1234567890",
                "recordId2                            some data here 98765432109876543210",
                "recordId2                            some data here",
                "recordId2                            some data here",
                "recordId2                            0987654321 0987654321 0987654321"
        );
        try (var mocked = mockStatic(Files.class)) {
            mocked.when(() -> Files.lines(Path.of(filePath))).thenReturn(lines);

            fileParsingService.parseAndSaveFile(filePath);

            verify(recordRepository, times(1)).saveAll(argThat(records -> {
                assertEquals(2, records.size());
                Record record1 = records.get(0);
                assertEquals("recordId1", record1.recordId());
                assertEquals(12345678901234567890.0, record1.marketValue());
                assertEquals(3703703700.0, record1.sum());

                Record record2 = records.get(1);
                assertEquals("recordId2", record2.recordId());
                assertEquals(98765432109876543210.0, record2.marketValue());
                assertEquals(2962962963.0, record2.sum());
                return true;
            }));
        }
    }

    @Test
    void testIncompleteChunkThrowsException() throws IOException {
        String filePath = "path/to/your/huge/file.txt";
        
        Stream<String> lines = Stream.of(
                "recordId1                            some data here 12345678901234567890",
                "recordId1                            some data here",
                "recordId1                            some data here"
        );
        try (var mocked = mockStatic(Files.class)) {
            mocked.when(() -> Files.lines(Path.of(filePath))).thenReturn(lines);

            IllegalStateException exception = assertThrows(IllegalStateException.class, () -> {
                fileParsingService.parseAndSaveFile(filePath);
            });

            assertEquals("File does not contain a multiple of 4 lines. Total lines: 3", exception.getMessage());
            verify(recordRepository, never()).saveAll(any());
        }
    }
}
```

### Summary

1. **Constants**: Defined in `FileParsingConstants` class to improve readability and maintainability.
2. **Service Class**: Updated to use constants for substring indices.
3. **Repository Class**: Remains unchanged.
4. **Main Class**: Remains unchanged.
5. **Tests**: Remain largely unchanged but ensure the integration with the repository is tested.

This refactoring makes the code more readable and easier to maintain by centralizing the substring indices into constants.
