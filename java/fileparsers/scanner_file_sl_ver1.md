Here's a Java program that meets your requirements for Java 11 or higher. It includes a service class for the main logic, a record for storing the data, constants for substring positions, and a main class to demonstrate the usage. I'll also provide an example of how to write unit tests for this service.

First, let's create the record to store our data:

```java
public record CusipData(String identifier, String value) {}
```

Now, let's create the service class with the main logic:

```java
import java.util.ArrayList;
import java.util.List;
import java.util.Scanner;

public class CusipDataService {
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

    public List<CusipData> processFile(Scanner scanner) throws IllegalStateException {
        List<CusipData> result = new ArrayList<>();
        
        while (scanner.hasNextLine()) {
            List<String> chunk = readChunk(scanner);
            if (chunk.size() != CHUNK_SIZE) {
                throw new IllegalStateException("Invalid chunk size. Expected " + CHUNK_SIZE + " but got " + chunk.size());
            }
            
            String cusip = chunk.get(0).substring(CUSIP_START, CUSIP_END);
            double marketValue = Double.parseDouble(chunk.get(0).substring(MARKET_VALUE_START, MARKET_VALUE_END).trim());
            
            double day7 = Double.parseDouble(chunk.get(3).substring(DAY_7_START, DAY_7_END).trim());
            double day8 = Double.parseDouble(chunk.get(3).substring(DAY_8_START, DAY_8_END).trim());
            double day9 = Double.parseDouble(chunk.get(3).substring(DAY_9_START, DAY_9_END).trim());
            
            if (day7 != 0 || day8 != 0 || day9 != 0) {
                double sum = day7 + day8 + day9;
                double calculatedValue = sum * marketValue;
                
                result.add(new CusipData(cusip, "1"));
                result.add(new CusipData(String.valueOf(marketValue), String.valueOf(calculatedValue)));
            }
        }
        
        return result;
    }

    private List<String> readChunk(Scanner scanner) {
        List<String> chunk = new ArrayList<>();
        for (int i = 0; i < CHUNK_SIZE; i++) {
            if (scanner.hasNextLine()) {
                chunk.add(scanner.nextLine());
            }
        }
        return chunk;
    }
}
```

Now, let's create a main class to demonstrate the usage:

```java
import java.io.File;
import java.io.FileNotFoundException;
import java.util.List;
import java.util.Scanner;

public class Main {
    public static void main(String[] args) {
        try (Scanner scanner = new Scanner(new File("path/to/your/file.txt"))) {
            CusipDataService service = new CusipDataService();
            List<CusipData> result = service.processFile(scanner);
            
            for (CusipData data : result) {
                System.out.println(data);
            }
        } catch (FileNotFoundException e) {
            System.err.println("File not found: " + e.getMessage());
        } catch (IllegalStateException e) {
            System.err.println("Error processing file: " + e.getMessage());
        }
    }
}
```

For unit tests, you can use JUnit 5. Here's an example of how to write some tests for the `CusipDataService` class:

```java
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import java.util.List;
import java.util.Scanner;

import static org.junit.jupiter.api.Assertions.*;

class CusipDataServiceTest {
    private CusipDataService service;

    @BeforeEach
    void setUp() {
        service = new CusipDataService();
    }

    @Test
    void testValidInput() {
        String input = 
            "          1234567890          100.00                                                                                           \n" +
            "          1234567890                                                                                                           \n" +
            "          1234567890                                                                                                           \n" +
            "          1234567890                                              10.00              20.00              30.00                 \n";
        Scanner scanner = new Scanner(input);
        
        List<CusipData> result = service.processFile(scanner);
        
        assertEquals(2, result.size());
        assertEquals(new CusipData("1234567890", "1"), result.get(0));
        assertEquals(new CusipData("100.00", "6000.0"), result.get(1));
    }

    @Test
    void testZeroValues() {
        String input = 
            "          1234567890          100.00                                                                                           \n" +
            "          1234567890                                                                                                           \n" +
            "          1234567890                                                                                                           \n" +
            "          1234567890                                               0.00               0.00               0.00                 \n";
        Scanner scanner = new Scanner(input);
        
        List<CusipData> result = service.processFile(scanner);
        
        assertTrue(result.isEmpty());
    }

    @Test
    void testInvalidChunkSize() {
        String input = 
            "          1234567890          100.00                                                                                           \n" +
            "          1234567890                                                                                                           \n" +
            "          1234567890                                                                                                           \n";
        Scanner scanner = new Scanner(input);
        
        assertThrows(IllegalStateException.class, () -> service.processFile(scanner));
    }
}
```

This program meets all the requirements you specified:

1. It processes a flat file with positional data.
2. It identifies the CUSIP from positions 11-20 and processes sets of 4 consecutive rows.
3. It extracts the market value from the first row, positions 104-116.
4. It extracts data for the 7th, 8th, and 9th days from the fourth row.
5. It ignores chunks where all three day values are 0.
6. It sums the 7th, 8th, and 9th day data.
7. It multiplies this sum with the market value.
8. It creates two `CusipData` objects for each valid chunk.
9. It throws an error if the remaining lines don't form a complete chunk.
10. All logic is in a service class.
11. Substrings are moved to constants.
12. Unit tests are provided.

To use this program, you would need to replace `"path/to/your/file.txt"` in the `Main` class with the actual path to your input file.
