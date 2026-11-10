# Sales-Analysis
let
    // 1. Load CSV file (Update the file path below to match your local machine)
    Source = Csv.Document(
        File.Contents("C:\Path\To\SalesData_1000Rows_WithIssues_copy.csv"),
        [Delimiter = ",", Columns = 11, Encoding = 65001, QuoteStyle = QuoteStyle.None]
    ),
    
    // 2. Promote first row to headers
    PromotedHeaders = Table.PromoteHeaders(Source, [PromoteAllScalars = true]),

    // 3. Standardize OrderDate (DD-MM-YYYY) using UK Locale & set initial column data types
    ChangedTypesWithLocale = Table.TransformColumnTypes(
        PromotedHeaders,
        {{"OrderDate", type date}},
        "en-GB"
    ),
    TypedColumns = Table.TransformColumnTypes(
        ChangedTypesWithLocale,
        {
            {"OrderID", Int64.Type},
            {"CustomerName", type text},
            {"Region", type text},
            {"Product", type text},
            {"Category", type text},
            {"Quantity", Int64.Type},
            {"UnitPrice", type number},
            {"Sales", type number},
            {"Cost", type number},
            {"Profit", type number}
        }
    ),

    // 4. Remove rows where OrderDate is missing (null) so time-series charts stay accurate
    FilteredValidDates = Table.SelectRows(TypedColumns, each [OrderDate] <> null),

    // 5. Remove duplicate rows based on OrderID
    RemovedDuplicates = Table.Distinct(FilteredValidDates, {"OrderID"}),

    // 6. Replace missing/blank text fields with "Unknown"
    CleanedTextColumns = Table.TransformColumns(
        RemovedDuplicates,
        {
            {"CustomerName", each if _ = null or Text.Trim(_) = "" then "Unknown" else _, type text},
            {"Region", each if _ = null or Text.Trim(_) = "" then "Unknown" else _, type text},
            {"Product", each if _ = null or Text.Trim(_) = "" then "Unknown" else _, type text},
            {"Category", each if _ = null or Text.Trim(_) = "" then "Unknown" else _, type text}
        }
    ),

    // 7. Impute missing numeric values using exact accounting relationships
    //    (Sales = Quantity * UnitPrice | Profit = Sales - Cost)
    FixedUnitPrice = Table.AddColumn(
        CleanedTextColumns,
        "CleanUnitPrice",
        each if [UnitPrice] = null and [Sales] <> null and [Quantity] <> null and [Quantity] <> 0
             then [Sales] / [Quantity]
             else [UnitPrice],
        type number
    ),
    FixedSales = Table.AddColumn(
        FixedUnitPrice,
        "CleanSales",
        each if [Sales] = null and [Quantity] <> null and [CleanUnitPrice] <> null
             then [Quantity] * [CleanUnitPrice]
             else [Sales],
        type number
    ),
    FixedCost = Table.AddColumn(
        FixedSales,
        "CleanCost",
        each if [Cost] = null and [CleanSales] <> null and [Profit] <> null
             then [CleanSales] - [Profit]
             else [Cost],
        type number
    ),
    FixedProfit = Table.AddColumn(
        FixedCost,
        "CleanProfit",
        each if [Profit] = null and [CleanSales] <> null and [CleanCost] <> null
             then [CleanSales] - [CleanCost]
             else [Profit],
        type number
    ),

    // 8. Remove incomplete original numeric columns and rename cleaned columns back
    RemovedOldNumericCols = Table.RemoveColumns(FixedProfit, {"UnitPrice", "Sales", "Cost", "Profit"}),
    FinalCleanedTable = Table.RenameColumns(
        RemovedOldNumericCols,
        {
            {"CleanUnitPrice", "UnitPrice"},
            {"CleanSales", "Sales"},
            {"CleanCost", "Cost"},
            {"CleanProfit", "Profit"}
        }
    )
in
    FinalCleanedTable
