

```java
// File: CusipRecord.java
public record CusipRecord(String cusip, BigDecimal marketValue, long count) {}

// File: CusipService.java
import java.io.File;
import java.io.FileNotFoundException;
import java.math.BigDecimal;
import java.util.ArrayList;
import java.util.List;
import java.util.Scanner;

public class CusipService {
    private static final int CUSIP_START = 10;
    private static final int CUSIP_END = 20;
    private static final int MARKET_VALUE_START = 103;
    private static final int MARKET_VALUE_END = 116;
    private static final int DAY_7_START = 134;
    private static final int DAY_7_END = 146;
    private static final int DAY_8_START = 148;
    private static final int DAY_8_END = 160;
    private static final int DAY_9_START = 162;
    private static final int DAY_9_END = 174;
    private static final int CHUNK_SIZE = 4;

    public List<CusipRecord> processFile(String filePath) throws FileNotFoundException, IllegalStateException {
        List<CusipRecord> records = new ArrayList<>();
        try (Scanner scanner = new Scanner(new File(filePath))) {
            while (scanner.hasNextLine()) {
                List<String> chunk = readChunk(scanner);
                if (chunk.size() != CHUNK_SIZE) {
                    throw new IllegalStateException("Invalid chunk size. Expected " + CHUNK_SIZE + " but got " + chunk.size());
                }
                processChunk(chunk, records);
            }
        }
        return records;
    }

    private List<String> readChunk(Scanner scanner) {
        List<String> chunk = new ArrayList<>();
        for (int i = 0; i < CHUNK_SIZE && scanner.hasNextLine(); i++) {
            chunk.add(scanner.nextLine());
        }
        return chunk;
    }

    private void processChunk(List<String> chunk, List<CusipRecord> records) {
        String cusip = chunk.get(0).substring(CUSIP_START, CUSIP_END);
        BigDecimal marketValue = new BigDecimal(chunk.get(0).substring(MARKET_VALUE_START, MARKET_VALUE_END).trim());

        long day7 = Long.parseLong(chunk.get(3).substring(DAY_7_START, DAY_7_END).trim());
        long day8 = Long.parseLong(chunk.get(3).substring(DAY_8_START, DAY_8_END).trim());
        long day9 = Long.parseLong(chunk.get(3).substring(DAY_9_START, DAY_9_END).trim());

        if (day7 != 0 || day8 != 0 || day9 != 0) {
            long sum = day7 + day8 + day9;
            BigDecimal totalValue = marketValue.multiply(BigDecimal.valueOf(sum));

            records.add(new CusipRecord(cusip, BigDecimal.ZERO, 1));
            records.add(new CusipRecord(cusip, totalValue, 0));
        }
    }
}

// File: Main.java
import java.io.FileNotFoundException;
import java.util.List;

public class Main {
    public static void main(String[] args) {
        CusipService service = new CusipService();
        try {
            List<CusipRecord> records = service.processFile("path/to/your/file.txt");
            for (CusipRecord record : records) {
                System.out.println(record);
            }
        } catch (FileNotFoundException e) {
            System.err.println("File not found: " + e.getMessage());
        } catch (IllegalStateException e) {
            System.err.println("Error processing file: " + e.getMessage());
        }
    }
}

// File: CusipServiceTest.java
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.io.TempDir;

import java.io.FileWriter;
import java.io.IOException;
import java.math.BigDecimal;
import java.nio.file.Path;
import java.util.List;

import static org.junit.jupiter.api.Assertions.*;

class CusipServiceTest {
    private CusipService cusipService;

    @BeforeEach
    void setUp() {
        cusipService = new CusipService();
    }

    @Test
    void testValidFile(@TempDir Path tempDir) throws IOException {
        Path file = tempDir.resolve("test.txt");
        try (FileWriter writer = new FileWriter(file.toFile())) {
            writer.write("          1234567890          1000.00\n");
            writer.write("          1234567890\n");
            writer.write("          1234567890\n");
            writer.write("          1234567890                                                                                                      100           200           300\n");
        }

        List<CusipRecord> records = cusipService.processFile(file.toString());

        assertEquals(2, records.size());
        assertEquals("1234567890", records.get(0).cusip());
        assertEquals(BigDecimal.ZERO, records.get(0).marketValue());
        assertEquals(1, records.get(0).count());
        assertEquals("1234567890", records.get(1).cusip());
        assertEquals(new BigDecimal("600000.00"), records.get(1).marketValue());
        assertEquals(0, records.get(1).count());
    }

    @Test
    void testInvalidChunkSize(@TempDir Path tempDir) throws IOException {
        Path file = tempDir.resolve("test.txt");
        try (FileWriter writer = new FileWriter(file.toFile())) {
            writer.write("          1234567890          1000.00\n");
            writer.write("          1234567890\n");
            writer.write("          1234567890\n");
        }

        assertThrows(IllegalStateException.class, () -> cusipService.processFile(file.toString()));
    }

    @Test
    void testZeroDayValues(@TempDir Path tempDir) throws IOException {
        Path file = tempDir.resolve("test.txt");
        try (FileWriter writer = new FileWriter(file.toFile())) {
            writer.write("          1234567890          1000.00\n");
            writer.write("          1234567890\n");
            writer.write("          1234567890\n");
            writer.write("          1234567890                                                                                                        0             0             0\n");
        }

        List<CusipRecord> records = cusipService.processFile(file.toString());

        assertTrue(records.isEmpty());
    }
}
```

This program includes the following components:

1. `CusipRecord`: A record class to store the CUSIP data.
2. `CusipService`: The main service class containing the logic for processing the file.
3. `Main`: A simple main class to demonstrate how to use the `CusipService`.
4. `CusipServiceTest`: A JUnit 5 test class with unit tests for the `CusipService`.

The `CusipService` class includes the following features:

- Constants for all substring positions.
- A method to process the entire file (`processFile`).
- Helper methods to read chunks and process individual chunks.
- Error handling for invalid chunk sizes.
- Logic to ignore chunks where the 7th, 8th, and 9th day data are all zero.
- Creation of two `CusipRecord` objects per valid chunk, as specified.

The unit tests cover the following scenarios:

1. Processing a valid file with non-zero day values.
2. Handling an invalid chunk size (throwing an `IllegalStateException`).
3. Ignoring a chunk with all zero day values.

To use this program, you'll need to:

1. Save each class in its own `.java` file.
2. Set up a Java project with JUnit 5 for running the tests.
3. Modify the file path in the `Main` class to point to your input file.
4. Run the `Main` class to process your file and see the results.

This implementation should meet all your requirements and provide a solid foundation for processing CUSIP data from positional flat files.
