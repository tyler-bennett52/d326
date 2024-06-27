



## A. Summarize one real-world written business report that can be created from the DVD Dataset from the “La on Demand Assessment Environment and DVD Database” attachment.
I want to determine sales my location. THis is a relevant business report as any business should want to know which locations are providing the most revenue, and which aren't doing very well comparitively. The summary table will be an average of the past twelve months sales data, the detailed table will be each's stores monthly sales figure.

### 1. Identify the specific fields that will be included in the detailed table and the summary table of the report.
Detailed Table:

    paymentId: Integer (likely a unique identifier)
    paymentAmount: Decimal (to represent currency accurately)
    storeId: Integer (to identify each store uniquely)
    DATE: Date (to record the transaction date)

Summary Table :

    AverageQuarterlySales: Decimal(9,2) (as specified in your description)
    storeId: Integer (to identify each store uniquely)

### 2. Describe the types of data fields used for the report.
Datatypes will be INTEGERS (mostly for IDs and Quarter), DECIMAL for currency and DATE for individual transaction tracking.

### 3. Identify at least two specific tables from the given dataset that will provide the data necessary for the detailed table section and the summary table section of the report.
The tables that have the information I need are staff, store, and payment. Stores for store_id, payments for payment_amount, and staff to to link payments with stores.

### 4. Identify at least one field in the detailed table section that will require a custom transformation with a user-defined function and explain why it should be transformed (e.g., you might translate a field with a value of N to No and Y to Yes).
The Date field should be converted into a quarter (1, 2, 3, 4), to allow for aggregation and averaging in that timeframe.

### 5. Explain the different business uses of the detailed table section and the summary table section of the report.
Understanding revenue by store is a key metric for identifying over/under-performing stores, allowing for management to better optimize costs by investing less in under performing locations, and allows for more efficient expansion, opting to create new stores that align more closely with over-performers than under-performers. This is one of the very first metrics a Dvd rental company would be looking at if they needed to cut store locations as well (sorry blockbusters!).

### 6. Explain how frequently your report should be refreshed to remain relevant to stakeholders.
Quarterly is best. Updating faster than that would make the most recent (in process) quarter look worse because it would have less days in it. Updating slower would result in stale data.

## B. Provide original code for function(s) in text format that perform the transformation(s) you identified in part A4.
CREATE OR REPLACE FUNCTION date_to_quarter(date_value DATE) 
RETURNS INTEGER AS $$
BEGIN
    RETURN EXTRACT(QUARTER FROM date_value);
END;
$$ LANGUAGE plpgsql;

## C. Provide original SQL code in a text format that creates the detailed and summary tables to hold your report table sections.

-- Create the detailed table
CREATE TABLE detailed_sales_report (
    id SERIAL PRIMARY KEY,
    payment_id INTEGER NOT NULL,
    payment_amount DECIMAL(10,2) NOT NULL,
    store_id INTEGER NOT NULL,
    payment_date DATE NOT NULL,
    quarter INTEGER NOT NULL
);

-- Create the summary table
CREATE TABLE summary_sales_report (
    id SERIAL PRIMARY KEY,
    store_id INTEGER NOT NULL,
    quarter INTEGER NOT NULL,
    year INTEGER NOT NULL,
    average_quarterly_sales DECIMAL(10,2) NOT NULL
);

-- Ensure only one of each quarter per year per store
ALTER TABLE summary_sales_report
ADD CONSTRAINT unique_store_quarter_year 
UNIQUE (store_id, quarter, year);

## D. Provide an original SQL query in a text format that will extract the raw data needed for the detailed section of your report from the source database.
INSERT INTO detailed_sales_report (payment_id, payment_amount, store_id, payment_date, quarter)
SELECT 
    p.payment_id,
    p.amount AS payment_amount,
    s.store_id,
    p.payment_date,
    date_to_quarter(p.payment_date) AS quarter
FROM 
    payment p
JOIN 
    staff st ON p.staff_id = st.staff_id
JOIN 
    store s ON st.store_id = s.store_id
WHERE 
    p.payment_date >= CURRENT_DATE - INTERVAL '12 months'
ORDER BY 
    s.store_id, p.payment_date;

## E. Provide original SQL code in a text format that creates a trigger on the detailed table of the report that will continually update the summary table as data is added to the detailed table.

-- First, let's create a function that our trigger will use
CREATE OR REPLACE FUNCTION update_summary_table()
RETURNS TRIGGER AS $$
BEGIN
    -- Delete existing summary data for the affected store, quarter, and year
    DELETE FROM summary_sales_report
    WHERE store_id = NEW.store_id
      AND quarter = NEW.quarter
      AND year = EXTRACT(YEAR FROM NEW.payment_date);

    -- Insert updated summary data
    INSERT INTO summary_sales_report (store_id, quarter, year, average_quarterly_sales)
    SELECT 
        store_id,
        quarter,
        EXTRACT(YEAR FROM payment_date) AS year,
        AVG(payment_amount) AS average_quarterly_sales
    FROM 
        detailed_sales_report
    WHERE 
        store_id = NEW.store_id
        AND quarter = NEW.quarter
        AND EXTRACT(YEAR FROM payment_date) = EXTRACT(YEAR FROM NEW.payment_date)
    GROUP BY 
        store_id, quarter, EXTRACT(YEAR FROM payment_date);

    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- Now, let's create the trigger
CREATE TRIGGER update_summary_after_insert
AFTER INSERT ON detailed_sales_report
FOR EACH ROW
EXECUTE FUNCTION update_summary_table();

## F. Provide an original stored procedure in a text format that can be used to refresh the data in both the detailed table and summary table. The procedure should clear the contents of the detailed table and summary table and perform the raw data extraction from part D.

CREATE OR REPLACE PROCEDURE refresh_sales_report()
LANGUAGE plpgsql
AS $$
BEGIN
    -- Disable the trigger temporarily to avoid unnecessary updates
    ALTER TABLE detailed_sales_report DISABLE TRIGGER update_summary_after_insert;

    -- Clear both tables
    TRUNCATE TABLE detailed_sales_report;
    TRUNCATE TABLE summary_sales_report;

    -- Re-populate the detailed table
    INSERT INTO detailed_sales_report (payment_id, payment_amount, store_id, payment_date, quarter)
    SELECT 
        p.payment_id,
        p.amount AS payment_amount,
        s.store_id,
        p.payment_date,
        date_to_quarter(p.payment_date) AS quarter
    FROM 
        payment p
    JOIN 
        staff st ON p.staff_id = st.staff_id
    JOIN 
        store s ON st.store_id = s.store_id
    WHERE 
        p.payment_date >= CURRENT_DATE - INTERVAL '12 months'
    ORDER BY 
        s.store_id, p.payment_date;

    -- Re-enable the trigger
    ALTER TABLE detailed_sales_report ENABLE TRIGGER update_summary_after_insert;

    -- Manually update the summary table
    INSERT INTO summary_sales_report (store_id, quarter, year, average_quarterly_sales)
    SELECT 
        store_id,
        quarter,
        EXTRACT(YEAR FROM payment_date) AS year,
        AVG(payment_amount) AS average_quarterly_sales
    FROM 
        detailed_sales_report
    GROUP BY 
        store_id, quarter, EXTRACT(YEAR FROM payment_date);

    COMMIT;
END;
$$;
Does this code look right to you?

### 1. Identify a relevant job scheduling tool that can be used to automate the stored procedure.
I like pgAgent for this tool. It would only need to run at the end of a quarter, so I would probably run it on the 3rd day of each quarter (to allow for refunds to process). I would probably run it outside of business hours to prevent any conflicts, so 0001 on the 3rd day of each quarter is what I'm thinking. 

## G. Provide a Panopto video recording that includes the presenter and a vocalized demonstration of the functionality of the code used for the analysis.

Note: For instructions on how to access and use Panopto, use the "Panopto How-To Videos" web link provided below. To access Panopto's website, navigate to the web link titled "Panopto Access," and then choose to log in using the �GU” option. If prompted, log in using your WGU student portal credentials, and then it will forward you to Panopto’s website.

�WTo submit your recording, upload it to the Panopto drop box titled “Advanced Data Management D191 | D326 (Student Creators) [assignments].” Once the recording has been uploaded and processed in Panopto's system, retrieve the URL of the recording from Panopto and copy and paste it into the Links option. Upload the remaining task requirements using the Attachments option.

## H. Acknowledge all utilized sources, including any sources of third-party code, using in-text citations and references. If no sources are used, clearly declare that no sources were used to support your submission.

No third party sources were used in this submission.

--## I. Demonstrate professional communication in the content and presentation of your submission.


