# Bank Details System - Low-Level Design Document

## 1. Requirement Summary

The Tour Details System maintains tour-related records in the `TOUR_DETAILS` database. It performs the following operations:

- Filtering records based on `GROUP_SIZE` and `PRICE_PER_HEAD`
- Handling null values in the `TOUR_GUIDE` field
- Calculating discounts based on `GROUP_SIZE` and `LANGUAGE`
- Computing final prices with discount tiers
- Generating output files in specific formats
- Updating and maintaining necessary records

## 2. Component List

### 2.1 Database Components

| Component Name | Description |
|---------------|------------|
| **TOUR_DETAILS** | Stores information on places, guides, languages, dates, group sizes, and prices |
| **SEASON_DISCOUNT** | Stores calculated discount details |

### 2.2 SPUFI Components

| Component Name | Description |
|---------------|------------|
| **SB12L** | Creates the `TOUR_DETAILS` table |
| **SB22L** | Creates the `SEASON_DISCOUNT` table |
| **SB32L** | Inserts sample values into `TOUR_DETAILS` |
| **SB42L** | Performs `LEFT OUTER JOIN` for records with `LANGUAGE = 'ENG'` and `GROUP_SIZE > 10` |
| **SB52L** | Fetches records from `SEASON_DISCOUNT` with `DISCOUNT <= 20` and converts them to lowercase |

### 2.3 COBOL Program Components

| Component Name | Description |
|---------------|------------|
| **CB12L** | Main program for processing tour details |

#### 2.3.1 Program Structure (CB12L)

- **MAIN-PROCEDURE** - Controls program flow
- **100-INITIALIZE** - Initializes variables and DB connection
- **200-PROCESS-RECORDS** - Main processing section
- **300-FETCH-RECORDS** - Fetches records from DB2
- **400-PROCESS-NULL-GUIDE** - Handles null `TOUR_GUIDE` values
- **500-CALCULATE-DISCOUNT** - Calculates discounts based on criteria
- **600-CALCULATE-FINAL-PRICE** - Computes final prices
- **700-INSERT-SEASON-DISCOUNT** - Inserts values into `SEASON_DISCOUNT`
- **800-WRITE-OUTPUT** - Writes records to output files
- **900-CLEANUP** - Closes files and DB connection

### 2.4 JCL Components

| Component Name | Description |
|---------------|------------|
| **JB12L** | Compiles and executes the COBOL program |

### 2.5 Output Files

| File DD Name | Record Length | Description |
|-------------|--------------|-------------|
| **OUTTOUR** | 80 | Stores processed records with discount info |
| **OUTODEL** | 80 | Stores records with null `TOUR_GUIDE` values |

---

## 3. Flowcharts

### System Process Flow

1. Create DB2 Tables:
   - `TOUR_DETAILS`
   - `SEASON_DISCOUNT`
2. Insert Sample Data into `TOUR_DETAILS`
3. Execute COBOL Program:
   - Initialize Program Variables
   - Open DB2 Connection
   - Fetch Records
   - Process Discounts and Final Prices
   - Write Outputs
   - Close DB2 Connection
4. Execute SPUFI Queries:
   - `LEFT OUTER JOIN`
   - Lowercase Conversion Query

---

## 4. Pseudo Code

### 4.1 Main Program (CB12L)

```cobol
PROCEDURE DIVISION

    PERFORM 100-INITIALIZE

    PERFORM 200-PROCESS-RECORDS UNTIL NO-MORE-RECORDS

    PERFORM 900-CLEANUP

    STOP RUN.

100-INITIALIZE

    OPEN OUTPUT FILES
    CONNECT TO DATABASE
    EXECUTE "WHENEVER SQLERROR GOTO ERROR-HANDLER"
    PREPARE CURSOR.

200-PROCESS-RECORDS

    PERFORM 300-FETCH-RECORDS
    IF SQLCODE = 100 THEN
        SET NO-MORE-RECORDS TO TRUE
    ELSE
        IF TOUR-GUIDE IS NULL THEN
            PERFORM 400-PROCESS-NULL-GUIDE
        ELSE
            PERFORM 500-CALCULATE-DISCOUNT
            PERFORM 600-CALCULATE-FINAL-PRICE
            PERFORM 700-INSERT-SEASON-DISCOUNT
            PERFORM 800-WRITE-OUTPUT
        END-IF
    END-IF.

300-FETCH-RECORDS

    EXEC SQL
        FETCH CURSOR INTO :TOUR-PLACE, :TOUR-GUIDE, :TOUR-GUIDE-NULL,  
        :LANGUAGE, :TOUR-DATE, :GROUP-SIZE, :PRICE-PER-HEAD
    END-EXEC.

400-PROCESS-NULL-GUIDE

    MOVE 'YTD' TO TOUR-GUIDE
    WRITE TO OUTODEL
    EXEC SQL
        DELETE FROM TOUR_DETAILS WHERE CURRENT OF CURSOR-NAME
    END-EXEC.

500-CALCULATE-DISCOUNT

    IF GROUP-SIZE = 5 THEN
        COMPUTE DISCOUNT = PRICE-PER-HEAD * 0.01
    ELSE IF GROUP-SIZE > 5 AND GROUP-SIZE <= 10 AND LANGUAGE = 'ENG' THEN
        COMPUTE DISCOUNT = PRICE-PER-HEAD * 0.02
    ELSE IF GROUP-SIZE > 10 AND GROUP-SIZE <= 20 THEN
        COMPUTE DISCOUNT = PRICE-PER-HEAD * 0.03
    ELSE IF GROUP-SIZE > 20 AND GROUP-SIZE <= 30 AND LANGUAGE = 'ENG' THEN
        COMPUTE DISCOUNT = PRICE-PER-HEAD * 0.018
    ELSE IF GROUP-SIZE > 20 AND GROUP-SIZE <= 30 AND LANGUAGE = 'TAM' THEN
        COMPUTE DISCOUNT = PRICE-PER-HEAD * 0.015
    END-IF.

600-CALCULATE-FINAL-PRICE

    COMPUTE GROUP-DISCOUNT = GROUP-SIZE * DISCOUNT

    IF DISCOUNT >= 10 AND DISCOUNT <= 20 THEN
        COMPUTE FINAL-PRICE = PRICE-PER-HEAD - (GROUP-SIZE * DISCOUNT) - 10
    ELSE IF DISCOUNT > 20 AND DISCOUNT <= 40 THEN
        COMPUTE FINAL-PRICE = PRICE-PER-HEAD - (GROUP-SIZE * DISCOUNT) - 12
    ELSE IF DISCOUNT > 40 AND DISCOUNT <= 50 THEN
        COMPUTE FINAL-PRICE = PRICE-PER-HEAD - (GROUP-SIZE * DISCOUNT) - 13
    ELSE IF DISCOUNT > 50 AND DISCOUNT < 100 THEN
        COMPUTE FINAL-PRICE = PRICE-PER-HEAD - (GROUP-SIZE * DISCOUNT) - 9
    END-IF.

700-INSERT-SEASON-DISCOUNT

    EXEC SQL
        INSERT INTO SEASON_DISCOUNT  
        (TOUR_PLACE, TOUR_GUIDE, TOUR_DATE, DISCOUNT, FINAL_PRICE, GROUP_DISCOUNT)
        VALUES (:TOUR-PLACE, :TOUR-GUIDE, :TOUR-DATE, :DISCOUNT, :FINAL-PRICE, :GROUP-DISCOUNT)
    END-EXEC.

800-WRITE-OUTPUT

    WRITE TO OUTTOUR.

900-CLEANUP

    CLOSE CURSOR
    DISCONNECT FROM DATABASE
    CLOSE FILES.
