# 1. 2025

## 1.1. Sales
- Sales `Carrefour` tanggal 01 Januari 2025 merupakan sales week 5 (week terakhir) dari bulan Desember 2024.
- Sales `Carrefour` tanggal 04 Februari 2025 merupakan sales week 5 (week terakhir) dari bulan Januari 2025
- Sales `Carrefour` tanggal 01 Maret 2025 merupakan sales week 5 (week terakhir) dari bulan Februari 2025
- Sales `Carrefour` tanggal 01 April 2025 merupakan sales week 5 (week terakhir) dari bulan Maret 2025
- Sales `Carrefour` tanggal 01 Mei 2025 merupakan sales week 5 (week terakhir) dari bulan April 2025

## 2.1 Query untuk audit report
```sql
--
-- Parameters
--
SET @store_name = UPPER(''); -- Masukan nama toko (Harus sesuai dengan J-Stock).
SET @date_selected = CURRENT_DATE; -- Masukan tanggal yyyy-mm-dd
SET @audit = 'ferdi'; -- Username audit


--
-- Periode Bulan: Ambil bulan ke-4 s.d. ke-2 dari tanggal saat ini
--
SET @start_month = DATE_FORMAT(DATE_SUB(@date_selected, INTERVAL 4 MONTH), '%Y-%m');
SET @end_month = DATE_FORMAT(DATE_SUB(@date_selected, INTERVAL 2 MONTH), '%Y-%m');


--
-- Ambil ID store berdasarkan nama
--
SET @store_id = (
    SELECT id
    FROM stock_db.master_store
    WHERE UPPER(store_name) = @store_name
    LIMIT 1
);


--
-- Ambil ID Group Store
--
SET @group_store_id = (
    SELECT store_group
    FROM stock_db.master_store
    WHERE UPPER(store_name) = @store_name
    LIMIT 1
);


--
-- Data Stock Audit Harian
--
SELECT
    ms.store_name AS 'Store Name',
    DATE(msd.input_time) AS 'SO Date',
    mr.region AS 'Region',
    msr.sub_region AS 'Sub Region',
    COUNT(msd.item) AS 'Total SKU',
    SUM(msd.stock) AS 'Total Qty',
    SUM(mdi.item_price_a * msd.stock) AS 'Total Gross'
FROM stock_db.master_stock_daily AS msd
INNER JOIN stock_db.master_store AS ms ON msd.store = ms.id
INNER JOIN stock_db.master_data_item AS mdi ON msd.item = mdi.item_code
LEFT JOIN stock_db.master_region AS mr ON ms.store_region = mr.id
LEFT JOIN stock_db.master_sub_region AS msr ON ms.store_sub_region = msr.id
WHERE msd.store = @store_id AND DATE(msd.input_time) = @date_selected AND msd.input_by = 'ferdi'
GROUP BY mr.region, msr.sub_region, ms.store_name, DATE(msd.input_time);


--
-- Rata-rata Sales 3 Bulan Terakhir (bulan ke-4 s.d. ke-2)
--
SELECT
    ROUND(SUM(total_item / 3), 0) AS 'Sales Avg 3 Month (SKU)',
    ROUND(SUM(total_qty / 3), 0) AS 'Sales Avg 3 Month (Qty)',
    ROUND(SUM(total_gross / 3), 0) AS 'Sales Avg 3 Month (Gross)'
FROM (
    SELECT
        COUNT(DISTINCT msr.item) AS total_item,
        COALESCE(SUM(msr.sales_real), 0) AS total_qty,
        COALESCE(SUM(mdi.item_price_a * msr.sales_real), 0) AS total_gross
    FROM stock_db.master_sales_real AS msr
    INNER JOIN stock_db.master_data_item AS mdi ON msr.item = mdi.item_code
    WHERE msr.store = @store_id
        AND DATE_FORMAT(msr.transaction_date, '%Y-%m') BETWEEN @start_month AND @end_month
) AS monthly_sales;


--
-- Data Stock Tanpa Penjualan dalam 3 Bulan Terakhir
--
SELECT
    COUNT(DISTINCT msd.item) AS 'Stock Item No Selling (SKU)',
    COALESCE(SUM(msd.stock), 0) AS 'Stock Item No Selling (Qty)',
    COALESCE(SUM(msd.stock * mdi.item_price_a), 0) AS 'Stock Item No Selling (Gross)'
FROM stock_db.master_stock_daily AS msd
LEFT JOIN stock_db.master_data_item AS mdi ON msd.item = mdi.item_code
LEFT JOIN (
    SELECT item AS item_code, SUM(sales_real) AS qty
    FROM stock_db.master_sales_real
    WHERE store = @store_id
        AND DATE_FORMAT(transaction_date, '%Y-%m') BETWEEN @start_month AND @end_month
    GROUP BY item
) AS sales ON msd.item = sales.item_code
WHERE msd.store = @store_id
    AND DATE(msd.input_time) = @date_selected
    AND msd.input_by = @audit
    AND sales.qty IS NULL;
    
    
--
-- Mengambil data item yang mmeiliki penjualan
-- tetapi tidak memiliki stock.
--
SELECT
	COUNT(DISTINCT msr.item) AS 'Item Selling No Stock (SKU)',
	COALESCE(SUM(msr.sales_real), 0) AS 'Item Selling No Stock (Qty)',
	COALESCE(SUM(msr.sales_real * mdi.item_price_a), 0) AS 'Item Selling No Stock (Gross)'
FROM stock_db.master_sales_real AS msr
LEFT JOIN stock_db.master_data_item AS mdi ON msr.item = mdi.item_code
LEFT JOIN (
	SELECT item AS item_code, SUM(stock) AS qty
	FROM stock_db.master_stock_daily
	WHERE store = @store_id
		AND DATE(input_time) = @date_selected
		AND input_by = @audit
	GROUP BY item
) AS msd ON msr.item = msd.item_code
WHERE msr.store = @store_id
	AND (DATE_FORMAT(msr.transaction_date, '%Y-%m') BETWEEN @start_month AND @end_month)
	AND msd.qty IS NULL;


--
-- Mengambil data item yang tidak memiliki stok tetapi
-- memiliki penjualan pada group store selama 3 bulan terakhir.
--
SELECT
	COUNT(DISTINCT sales.item_code) AS 'Item Selling Group Store No Stock (SKU)',
	COALESCE(SUM(sales.qty), 0) AS 'Item Selling Group Store No Stock (Qty)',
	COALESCE(SUM(sales.qty * item.price), 0) AS 'Item Selling Group Store No Stock (Gross)'
FROM (
	SELECT item AS item_code, SUM(sales_real) AS qty
	FROM stock_db.master_sales_real
	WHERE DATE_FORMAT(transaction_date, '%Y-%m') BETWEEN @start_month AND @end_month
		AND store IN (
			SELECT id
			FROM stock_db.master_store
			WHERE store_group = @group_store_id
		)
	GROUP BY item
) AS sales
LEFT JOIN (
	SELECT item AS item_code, SUM(stock) AS qty
	FROM stock_db.master_stock_daily
	WHERE store = @store_id
		AND DATE(input_time) = @date_selected
		AND input_by = @audit
	GROUP BY item
) AS stock ON sales.item_code = stock.item_code
LEFT JOIN (
	SELECT DISTINCt item_code, CONVERT(item_price_a, SIGNED INTEGER) AS price
	FROM stock_db.master_data_item
) AS item ON sales.item_code = item.item_code
WHERE stock.qty IS NULL;
```
