# 1. 2025

## 1.1. Sales
- Sales `Carrefour` tanggal 01 Januari 2025 merupakan sales week 5 (week terakhir) dari bulan Desember 2024.
- Sales `Carrefour` tanggal 04 Februari 2025 merupakan sales week 5 (week terakhir) dari bulan Januari 2025
- Sales `Carrefour` tanggal 01 Maret 2025 merupakan sales week 5 (week terakhir) dari bulan Februari 2025
- Sales `Carrefour` tanggal 01 April 2025 merupakan sales week 5 (week terakhir) dari bulan Maret 2025
- Sales `Carrefour` tanggal 01 Mei 2025 merupakan sales week 5 (week terakhir) dari bulan April 2025
- Sales `Carrefour` tanggal 01 Juni 2025 merupakan sales week 5 (week terakhir) dari bulan Mei 2025
- Sales `Carrefour` tanggal 01 Juli 2025 merupakan sales week 5 (week terakhir) dari bulan Juni 2025

## 2.1 Query untuk audit report
```sql
--
-- Parameters
--
SET @store_name = UPPER(''); -- Masukan nama toko (Harus sesuai dengan J-Stock).
SET @date_selected = CURRENT_DATE; -- Masukan tanggal SO (format: yyyy-mm-dd)
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


SELECT
    -- 1. Data Stock Audit Harian
    daily_stock.store_name AS 'Store Name',
    daily_stock.so_date AS 'SO Date',
    daily_stock.region_name AS 'Region',
    daily_stock.sub_region_name AS 'Sub Region',
    daily_stock.sku AS 'Stock (SKU)',
    daily_stock.qty AS 'Stock (Qty)',
    daily_stock.gross AS 'Stock (Gross)',
    
    
    -- 2. Estimasi sales.
    estimasi_sales.sku AS 'Estimasi Sales (SKU)',
    estimasi_sales.qty AS 'Estimasi Sales (Qty)',
    estimasi_sales.gross AS 'Estimasi Sales (Gross)',

    -- 3. Rata-rata Sales 3 Bulan Terakhir
    sales_avg.sku AS 'Sales Avg (SKU)',
    sales_avg.qty AS 'Sales Avg (Qty)',
    sales_avg.gross AS 'Sales Avg (Gross)',

    -- 4. Stock Tanpa Penjualan
    stock_no_sales.sku AS 'Stock No Sales (SKU)',
    stock_no_sales.qty AS 'Stock No Sales (Qty)',
    stock_no_sales.gross AS 'Stock No Sales (Gross)',

    -- 5. Item Terjual tapi Tidak Ada Stock
    sales_no_stock.sku AS 'Sales No Stock (SKU)',
    sales_no_stock.qty AS 'Sales No Stock (Qty)',
    sales_no_stock.gross AS 'Sales No Stock (Gross)',

    -- 6. Item Terjual di Group Store tapi Tidak Ada Stock
    sales_group_store_no_stock.sku AS 'Sales Group Store No Stock (SKU)',
    sales_group_store_no_stock.qty AS 'Sales Group Store No Stock (Qty)',
    sales_group_store_no_stock.gross AS 'Sales Group Store No Stock (Gross)'

--
-- 1. Mengambil data stock hasil audit
--
FROM (
    SELECT
        UPPER(ms.store_name) AS store_name,
        DATE(msd.input_time) AS so_date,
        mr.region AS region_name,
        msr.sub_region AS sub_region_name,
        COUNT(msd.item) AS sku,
        SUM(msd.stock) AS qty,
        SUM(mdi.item_price_a * msd.stock) AS gross
    FROM stock_db.master_stock_daily AS msd
    INNER JOIN stock_db.master_store AS ms ON msd.store = ms.id
    LEFT JOIN stock_db.master_data_item AS mdi ON msd.item = mdi.item_code
    LEFT JOIN stock_db.master_region AS mr ON ms.store_region = mr.id
    LEFT JOIN stock_db.master_sub_region AS msr ON ms.store_sub_region = msr.id
    WHERE msd.store = @store_id AND DATE(msd.input_time) = @date_selected AND msd.input_by = @audit
    GROUP BY mr.region, msr.sub_region, ms.store_name, DATE(msd.input_time)
) AS daily_stock


--
-- 2. Estimasi sales.
--
CROSS JOIN (
	SELECT
		COUNT(sku) AS sku,
		SUM(qty) AS qty,
		SUM(gross) AS gross
	FROM (
		SELECT
			master_item.item_code AS sku,
			SUM((COALESCE(stock_awal.qty, 0) + COALESCE(surat_jalan.qty, 0) - COALESCE(retur.qty, 0)) - COALESCE(audit_stock.qty, 0)) AS qty,
			SUM(master_item.item_price_a * ((COALESCE(stock_awal.qty, 0) + COALESCE(surat_jalan.qty, 0) - COALESCE(retur.qty, 0)) - COALESCE(audit_stock.qty, 0))) AS gross
		FROM stock_db.master_data_item AS master_item
		
		-- Stock awal
		LEFT JOIN (
			SELECT sam.code AS item_code, SUM(sam.stock) AS qty
			FROM stock_db.stock_awal_monthly AS sam
			WHERE sam.store = @store_id
				AND sam.`year` = YEAR(@date_selected)
				AND sam.`month` = MONTH(@date_selected)
			GROUP BY sam.code
		) AS stock_awal ON master_item.item_code = stock_awal.item_code
		
		-- Surat jalan
		LEFT JOIN (
			SELECT item AS item_code, SUM(qty) AS qty
			FROM stock_db.master_sj_daily
			WHERE store = @store_id
				AND DATE(input_time)
					BETWEEN DATE_FORMAT(@date_selected, '%Y-%m-%01')
					AND DATE_SUB(@date_selected, INTERVAL 1 DAY)
			GROUP BY item
		) AS surat_jalan ON master_item.item_code = surat_jalan.item_code
		
		-- Return
		LEFT JOIN (
			SELECT item AS item_code, SUM(qty) AS qty
			FROM stock_db.master_retur_daily
			WHERE store = @store_id
				AND DATE(input_time)
					BETWEEN DATE_FORMAT(@date_selected, '%Y-%m-%01')
					AND DATE_SUB(@date_selected, INTERVAL 1 DAY)
			GROUP BY item
		) retur ON master_item.item_code = retur.item_code
		
		-- Audit stock
		LEFT JOIN (
			SELECT item AS item_code, SUM(stock) AS qty
			FROM stock_db.master_stock_daily
			WHERE store = @store_id
				AND DATE(input_time) = @date_selected
				AND input_by = @audit
			GROUP BY item
		) AS audit_stock ON master_item.item_code = audit_stock.item_code
		
		GROUP BY master_item.item_code
		HAVING qty <> 0
		ORDER BY master_item.item_code ASC
	) AS estimasi_sales
) AS estimasi_sales


--
-- 3. Rata-rata Sales 3 Bulan Terakhir
-- Contoh: Jika saat ini bulan mei maka ambil data sales dari Januari s.d. Maret.
--
CROSS JOIN (
    SELECT
        ROUND(SUM(total_item / 3), 0) AS sku,
        ROUND(SUM(total_qty / 3), 0) AS qty,
        ROUND(SUM(total_gross / 3), 0) AS gross
    FROM (
        SELECT
            COUNT(DISTINCT msr.item) AS total_item,
            COALESCE(SUM(msr.sales_real), 0) AS total_qty,
            COALESCE(SUM(mdi.item_price_a * msr.sales_real), 0) AS total_gross
        FROM stock_db.master_sales_real AS msr
        INNER JOIN stock_db.master_data_item AS mdi ON msr.item = mdi.item_code
        WHERE msr.store = @store_id AND DATE_FORMAT(msr.transaction_date, '%Y-%m') BETWEEN @start_month AND @end_month
    ) AS monthly_sales
) AS sales_avg


--
-- 4. Data Stock Tanpa Penjualan dalam 3 Bulan Terakhir
-- Contoh: Jika saat ini bulan mei maka ambil data sales dari Januari s.d. Maret.
--
CROSS JOIN (
    SELECT
        COUNT(DISTINCT msd.item) AS sku,
        COALESCE(SUM(msd.stock), 0) AS qty,
        COALESCE(SUM(msd.stock * mdi.item_price_a), 0) AS gross
    FROM stock_db.master_stock_daily AS msd
    LEFT JOIN stock_db.master_data_item AS mdi ON msd.item = mdi.item_code
    LEFT JOIN (
        SELECT item AS item_code, SUM(sales_real) AS qty
        FROM stock_db.master_sales_real
        WHERE store = @store_id AND DATE_FORMAT(transaction_date, '%Y-%m') BETWEEN @start_month AND @end_month
        GROUP BY item
    ) AS sales ON msd.item = sales.item_code
    WHERE msd.store = @store_id
        AND DATE(msd.input_time) = @date_selected
        AND msd.input_by = @audit
        AND sales.qty IS NULL
) AS stock_no_sales


--
-- 5. Sales 3 bulan yang tidak memiliki stock
-- Contoh: Jika saat ini bulan mei maka ambil data sales dari Januari s.d. Maret.
--
CROSS JOIN (
    SELECT
        COUNT(DISTINCT msr.item) AS sku,
        COALESCE(SUM(msr.sales_real), 0) AS qty,
        COALESCE(SUM(msr.sales_real * mdi.item_price_a), 0) AS gross
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
        AND msd.qty IS NULL
) AS sales_no_stock


--
-- 6. Sales by group store 3 bulan yang tidak memiliki stock.
-- Contoh: Jika saat ini bulan mei maka ambil data sales dari Januari s.d. Maret.
--
CROSS JOIN (
    SELECT
        COUNT(DISTINCT sales.item_code) AS sku,
        COALESCE(SUM(sales.qty), 0) AS qty,
        COALESCE(SUM(sales.qty * item.price), 0) AS gross
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
        SELECT DISTINCT item_code, CONVERT(item_price_a, SIGNED INTEGER) AS price
        FROM stock_db.master_data_item
    ) AS item ON sales.item_code = item.item_code

    WHERE stock.qty IS NULL
) AS sales_group_store_no_stock;
```

## 2.2 Query untuk mengambil rekomendasi stok awal
```sql
-- Cukup ubah tahun dan bulan.
SET @tanggal = '2025-07-01';

SELECT
  UPPER(region.name) AS 'Region',
  UPPER(sub_region.sub_region) AS 'Sub Region',
  UPPER(store.store_name) AS 'Store Name',
  SUM(CASE WHEN stock.so_type = 2 AND stock.so_date = DATE_SUB(@tanggal, INTERVAL 1 DAY) THEN stock.total ELSE 0 END) AS 'Stok Pulang 30 Juni 2025',
  SUM(CASE WHEN stock.so_type = 1 AND stock.so_date = DATE_SUB(@tanggal, INTERVAL 0 DAY) THEN stock.total ELSE 0 END) AS 'Stock Masuk 01 Juli 2025',
  SUM(CASE WHEN stock.so_type = 2 AND stock.so_date = DATE_SUB(@tanggal, INTERVAL 0 DAY) THEN stock.total ELSE 0 END) AS 'Stock Pulang 01 Juli 2025',
  SUM(CASE WHEN stock.so_type = 1 AND stock.so_date = DATE_SUB(@tanggal, INTERVAL -1 DAY) THEN stock.total ELSE 0 END) AS 'Stock Masuk 02 Juli 2025',
  SUM(CASE WHEN stock.so_type = 2 AND stock.so_date = DATE_SUB(@tanggal, INTERVAL -1 DAY) THEN stock.total ELSE 0 END) AS 'Stock Pulang 02 Juli 2025',
  SUM(CASE WHEN stock.so_type = 1 AND stock.so_date = DATE_SUB(@tanggal, INTERVAL -2 DAY) THEN stock.total ELSE 0 END) AS 'Stock Masuk 03 Juli 2025',
  SUM(CASE WHEN stock.so_type = 2 AND stock.so_date = DATE_SUB(@tanggal, INTERVAL -2 DAY) THEN stock.total ELSE 0 END) AS 'Stock Pulang 03 Juli 2025',
  SUM(CASE WHEN stock.so_type = 1 AND stock.so_date = DATE_SUB(@tanggal, INTERVAL -3 DAY) THEN stock.total ELSE 0 END) AS 'Stock Masuk 04 Juli 2025',
  SUM(CASE WHEN stock.so_type = 2 AND stock.so_date = DATE_SUB(@tanggal, INTERVAL -3 DAY) THEN stock.total ELSE 0 END) AS 'Stock Pulang 04 Juli 2025',
  SUM(CASE WHEN stock.so_type = 1 AND stock.so_date = DATE_SUB(@tanggal, INTERVAL -4 DAY) THEN stock.total ELSE 0 END) AS 'Stock Masuk 05 Juli 2025',
  SUM(CASE WHEN stock.so_type = 2 AND stock.so_date = DATE_SUB(@tanggal, INTERVAL -4 DAY) THEN stock.total ELSE 0 END) AS 'Stock Pulang 05 Juli 2025',
  SUM(CASE WHEN stock.so_type = 1 AND stock.so_date = DATE_SUB(@tanggal, INTERVAL -5 DAY) THEN stock.total ELSE 0 END) AS 'Stock Masuk 06 Juli 2025',
  SUM(CASE WHEN stock.so_type = 2 AND stock.so_date = DATE_SUB(@tanggal, INTERVAL -5 DAY) THEN stock.total ELSE 0 END) AS 'Stock Pulang 06 Juli 2025',
  SUM(CASE WHEN stock.so_type = 1 AND stock.so_date = DATE_SUB(@tanggal, INTERVAL -6 DAY) THEN stock.total ELSE 0 END) AS 'Stock Masuk 07 Juli 2025',
  SUM(CASE WHEN stock.so_type = 2 AND stock.so_date = DATE_SUB(@tanggal, INTERVAL -6 DAY) THEN stock.total ELSE 0 END) AS 'Stock Pulang 07 Juli 2025',
  SUM(CASE WHEN stock.so_type = 1 AND stock.so_date = DATE_SUB(@tanggal, INTERVAL -7 DAY) THEN stock.total ELSE 0 END) AS 'Stock Masuk 08 Juli 2025',
  SUM(CASE WHEN stock.so_type = 2 AND stock.so_date = DATE_SUB(@tanggal, INTERVAL -7 DAY) THEN stock.total ELSE 0 END) AS 'Stock Pulang 08 Juli 2025',
  SUM(CASE WHEN stock.so_type = 1 AND stock.so_date = DATE_SUB(@tanggal, INTERVAL -8 DAY) THEN stock.total ELSE 0 END) AS 'Stock Masuk 09 Juli 2025',
  SUM(CASE WHEN stock.so_type = 2 AND stock.so_date = DATE_SUB(@tanggal, INTERVAL -8 DAY) THEN stock.total ELSE 0 END) AS 'Stock Pulang 09 Juli 2025',
  SUM(CASE WHEN stock.so_type = 1 AND stock.so_date = DATE_SUB(@tanggal, INTERVAL -9 DAY) THEN stock.total ELSE 0 END) AS 'Stock Masuk 10 Juli 2025',
  SUM(CASE WHEN stock.so_type = 2 AND stock.so_date = DATE_SUB(@tanggal, INTERVAL -9 DAY) THEN stock.total ELSE 0 END) AS 'Stock Pulang 10 Juli 2025'
FROM master_store AS store

LEFT JOIN (
  SELECT id, region AS name
  FROM master_region
) AS region ON store.store_region = region.id

LEFT JOIN (
  SELECT id, sub_region
  FROM master_sub_region
) AS sub_region ON store.store_sub_region = sub_region.id

LEFT JOIN (
  SELECT
    store AS store_id,
    DATE(input_time) AS so_date,
    type_so AS so_type,
    CAST(SUM(stock) AS UNSIGNED) AS total
  FROM master_stock_daily
  WHERE DATE(input_time) BETWEEN DATE_SUB(@tanggal, INTERVAL 1 DAY) AND DATE_SUB(@tanggal, INTERVAL -9 DAY)
  GROUP BY store, DATE(input_time), type_so
) AS stock ON store.id = stock.store_id

WHERE store.isActive = 1
GROUP BY region.name, sub_region.sub_region, store.store_name
ORDER BY region.name, store.store_name ASC;
```

