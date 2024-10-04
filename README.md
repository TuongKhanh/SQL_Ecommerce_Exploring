# SQL_Ecommerce_Exploring

  1. [Introduction and Motivation](#1-introduction-and-motivation)
  2. [The goal of creating this project](#2-the-goal-of-creating-this-project)
  3. [Import raw data](#3-import-raw-data)
  4. [Clear data](#4-clear-data)
  5. [Data Processing & Exploratory Data Analysis](#5-data-processing--exploratory-data-analysis)
  6. [Ask questions and solve it](#6-ask-questions-and-solve-it)

## 1. Introduction and Motivation

This file contains behavior data for 3 months (from 10-2019 to 12-2019) from a large multi-category online store.

Each row in the file represents an event. All events are related to products and users. Each event is like many-to-many relation between products and users.

Data collected by Open CDP project.

## 2. The goal of creating this project
 
--Làm sạch lại dữ liệu

	DECLARE @TableName NVARCHAR(255);
	DECLARE @Sql NVARCHAR(MAX);
	
 	DECLARE table_cursor CURSOR FOR 
	SELECT TABLE_NAME 
	FROM INFORMATION_SCHEMA.COLUMNS 
	WHERE COLUMN_NAME = 'event_time';
	OPEN table_cursor;
	FETCH NEXT FROM table_cursor INTO @TableName;

	WHILE @@FETCH_STATUS = 0
	BEGIN
    	SET @Sql = 'UPDATE ' + QUOTENAME(@TableName) + 
                ' SET event_time = REPLACE(event_time, '' UTC'', '''')' +
                ' WHERE event_time LIKE ''%UTC%''';  -- Chỉ cập nhật các dòng có chữ "UTC"
    
    	EXEC sp_executesql @Sql;  -- Thực hiện câu lệnh động
    	FETCH NEXT FROM table_cursor INTO @TableName;
	END
	CLOSE table_cursor
	DEALLOCATE table_cursor

--Tạo bảng

	create table eCommerce_table
	(
		 event_time		datetime
		,event_type		varchar(20)
		,product_id		bigint
		,category_id	bigint
		,category_code	varchar(100)
		,brand			varchar(50)
		,price			float
		,[user_id]		int
		,user_session	varchar(100)
	)

--MERGE CÁC BẢNG LẠI VỚI NHAU VÀ INSRET VÀO BẢNG eCommerce_table

	INSERT INTO  eCommerce_table
	SELECT * FROM "2019-Dec"
	UNION ALL
	SELECT * FROM "2019-Nov"
	UNION ALL
	SELECT * FROM "2019-Oct"


-- TÍNH SỐ LƯỢNG KHÁCH HÀNG TRONG TỪNG THÁNG VÀ TỈ LỆ PHẦN TRĂM VỚI QUÝ 4

	WITH NUMBER_CUSTOMER_Q4 AS
		(
			SELECT 
				COUNT(DISTINCT [user_id]) NUMBER_CUS_Q4
			FROM eCommerce_table
		)
			SELECT 
				 EVERY_MONTH 	 = CONVERT(nvarchar(7),ECT.event_time, 120)				
				,NUMBER_CUSTOMER = COUNT(DISTINCT [user_id])								
				,CUSTOMER_RATIO	 = COUNT(DISTINCT [user_id]) *100.0 /ncq.NUMBER_CUS_Q4	
			FROM eCommerce_table ECT
			CROSS JOIN NUMBER_CUSTOMER_Q4 ncq
			GROUP BY CONVERT(nvarchar(7),event_time, 120), ncq.NUMBER_CUS_Q4
			ORDER BY 1

/*
 Bảng này cho só lượng thấy khách hàng vào 3 tháng 10, 11,12 và tỉ lệ khách hàng từ tháng theo quý
 Số lượng khách hàng tăng nhẹ ở tháng 11 và giảm ở tháng 12
*/


--Số lượng giao dịch trung bình trên mỗi khách hàng theo từng tháng

	WITH TRANSACTION_CUSTOMERS AS
		(
			SELECT 
				 MONTH_OF_YEAR		= CONVERT(nvarchar(7),event_time, 120)	
				,NUMBER_TRANSACTION = COUNT(event_type)
			FROM eCommerce_table
			WHERE event_type ='purchase'
			GROUP BY CONVERT(nvarchar(7),event_time, 120)
		),
	CUSTOMER_ACT AS
		(
			SELECT 
				 MONTH_OF_YEAR	 = CONVERT(nvarchar(7),event_time, 120)	
				,NUMBER_CUSTOMER = COUNT(DISTINCT [user_id])
			FROM eCommerce_table
			WHERE event_type ='purchase'
			GROUP BY CONVERT(nvarchar(7),event_time, 120)
		)
	SELECT 
		 TCS.MONTH_OF_YEAR
		,TCS.NUMBER_TRANSACTION
		,CUS.NUMBER_CUSTOMER
		,AVG_TOTAL_TRANSACTION_PER_USER = ROUND(CAST(TCS.NUMBER_TRANSACTION AS FLOAT) / CUS.NUMBER_CUSTOMER,2)
	FROM		TRANSACTION_CUSTOMERS TCS
	INNER JOIN	CUSTOMER_ACT		  CUS ON TCS.MONTH_OF_YEAR = CUS.MONTH_OF_YEAR
	ORDER BY 1

/*
Bảng này cho thấy tổng số giao dịch trung bình trên mỗi người dùng trong Q4 2109. Người dùng điển hình đã thực hiện trung bình khoảng 135 giao dịch. Điều này có thể hữu ích để hiểu hành vi của người dùng, 
theo dõi mức độ tương tác của người dùng với nền tảng của bạn hoặc đánh giá hiệu quả của các chiến dịch tiếp thị hoặc khuyến mãi trong tháng cụ thể đó.
*/

-- số tiền khách hàng giao dịch trung bình trên mỗi đơn

	SELECT 
		 MONTH_OF_YEAR		= CONVERT(nvarchar(7),event_time, 120)
		,AMOUNT_TRANSACTION = SUM(price) 
		,NUMBER_CUSTOMER = COUNT(DISTINCT [user_id])
		,AVG_REVENUE_PER_USER_ACT = ROUND(SUM(price) /COUNT(DISTINCT [user_id]),3)
	FROM eCommerce_table
	WHERE event_type ='purchase'
	GROUP BY CONVERT(nvarchar(7),event_time, 120)

/*
bảng này cho thấy khách hàng tăng nhưng lượng dần qua từng tháng nhưng số lượng giao dịch lẫn doanh thu giảm

*/

	WITH CUSTOMER_ACT AS
		(
			SELECT 
				 MONTH_OF_YEAR		 = CONVERT(nvarchar(7),event_time, 120)					   
				,NUMBER_CUSTOMERS    = COUNT(DISTINCT [user_id])								   
				,NUMBER_VIEW		 = SUM(CASE WHEN event_type ='view'		THEN 1 ELSE 0 END) 
				,NUMBER_CART		 = SUM(CASE WHEN event_type ='cart'		THEN 1 ELSE 0 END) 
				,NUMBER_PURCHASE 	 = SUM(CASE WHEN event_type ='purchase'	THEN 1 ELSE 0 END) 
			FROM eCommerce_table
			GROUP BY (CONVERT(nvarchar(7),event_time, 120))
		),
	REVENUE_MONTH AS
		(
			SELECT 
				 MONTH_OF_YEAR	= CONVERT(nvarchar(7),event_time, 120)
				,TOTAL_PRICE	= SUM(price)
			FROM eCommerce_table
			WHERE event_type ='purchase'
			GROUP BY CONVERT(nvarchar(7),event_time, 120)
		)
	SELECT 
		 CUS.MONTH_OF_YEAR
		,CUS.NUMBER_CUSTOMERS
		,CUS.NUMBER_VIEW	
		,CUS.NUMBER_CART	
		,CUS.NUMBER_PURCHASE
		,AVG_OF_VIEWED			= CASE WHEN CUS.NUMBER_VIEW		> 0 THEN ROUND((CAST(CUS.NUMBER_VIEW		AS FLOAT) / CUS.NUMBER_CUSTOMERS)		,2)	ELSE 0 END 
		,RATING_OF_CART 		= CASE WHEN CUS.NUMBER_CART		> 0 THEN ROUND((CAST(CUS.NUMBER_CART		AS FLOAT) / CUS.NUMBER_CUSTOMERS) * 100 ,2) ELSE 0 END 
		,RATING_OF_PURCHASE 	= CASE WHEN CUS.NUMBER_PURCHASE	> 0 THEN ROUND((CAST(CUS.NUMBER_PURCHASE	AS FLOAT) / CUS.NUMBER_CUSTOMERS) * 100 ,2) ELSE 0 END 
		,REV.TOTAL_PRICE
	FROM		CUSTOMER_ACT	CUS
	LEFT JOIN	REVENUE_MONTH	REV ON CUS.MONTH_OF_YEAR = REV.MONTH_OF_YEAR
	ORDER BY 1

/*
Bảng này cho thấy số lượng 1 khách hàng xem sản phẩm, tỉ lệ phần trăm thêm vào giỏ hàng, tỉ lệ mua hàng và doanh thu theo tháng 
Số lượng khách hàng xem lại giảm nhưng cho vào giỏ hàng lại tăng đột biến tháng 12 nhưng thanh toán cho sản phẩm lại ko cao. 
Doanh thu tháng 12 lại thấp hơn so với 2 tháng trước điều này cho thấy sự chuyển đổi mua hàng không cao.

Doanh nghiệp cần tăng sự thu hút thiết kế và nội dung để tăng lượt xem. Sự chuyển đổi từ giỏ hàng sang mua hàng thấp, 
cần xem xét lại dịch vụ vận chuyển hay khả năng thanh toán có gây rắc rối cho khách hàng ko? 
*/


	WITH CUSTOMER_PURCHASED AS -- TÍNH SỐ LƯỢNG KHÁCH HÀNG MỚI VÀ TỈ LỆ GIỮ CHÂN KHÁCH HÀNG QUA TỪNG THÁNG 
		(
			SELECT 
				 [user_id]
				,SUM(price)		TOTAL_PRICE
				,FIRST_MONTH = MIN(CONVERT(nvarchar(7),event_time, 120))  
			FROM eCommerce_table
			WHERE event_type = 'purchase'
			GROUP BY [user_id]
		),
	NEW_CUSTOMER AS
		(
			SELECT 
				 FIRST_MONTH
				,NEW_CUSTOMER = COUNT([user_id])	
			FROM CUSTOMER_PURCHASED
			GROUP BY FIRST_MONTH
		),
	RETAINED_CUSTOMER AS
		(
			SELECT  
				[user_id]
				,RETAINTION_CUSTOMER = CONVERT(nvarchar(7),event_time, 120)  
			FROM eCommerce_table
			WHERE event_type ='purchase'
			GROUP BY 
				[user_id]
				,CONVERT(nvarchar(7),event_time, 120)
		),
	STT_CUSTOMER AS
		(
			SELECT 
			     NCS.FIRST_MONTH
			    ,RCS.RETAINTION_CUSTOMER
			    ,RETAINED_COUNT = COUNT(RCS.[user_id])  
			FROM RETAINED_CUSTOMER RCS
			LEFT JOIN CUSTOMER_PURCHASED NCS ON RCS.[user_id] = NCS.[user_id]
			GROUP BY 
			     NCS.FIRST_MONTH
			    ,RCS.RETAINTION_CUSTOMER
		)
	SELECT 
		 NCS.FIRST_MONTH
		,RIGHT(STT.RETAINTION_CUSTOMER,2) "MONTH"
		,STT.RETAINTION_CUSTOMER
		,NCS.NEW_CUSTOMER
		,STT.RETAINED_COUNT
		,RETENTION_RATE		= ROUND(CAST(STT.RETAINED_COUNT AS FLOAT)/NCS.NEW_CUSTOMER * 100,2)
	FROM STT_CUSTOMER STT 
	LEFT JOIN NEW_CUSTOMER NCS ON STT.FIRST_MONTH = NCS.FIRST_MONTH
	ORDER BY 1,2 
	GO

/*
 Số lượng khách hàng mới tăng nhẹ theo từng tháng điều này cho thấy chương trình dành 
 cho khách hàng mới có sự thu hút cao 
 Tỉ lệ khách hàng quay lại thấp cho thấy 
   - Cần xem xét lại chất lượng sản phẩm của nhà bán hàng
   - Đưa ra chương trình cho khách hàng thân thiết
   - Đảm bảo khách hàng có thể dễ dàng tiếp cận hỗ trợ khi họ gặp vấn đề
   - Thu thập phản hồi từ đánh giá của khách hàng để xem xét vấn đề ở sản phẩmm, giao hàng hay trải nghiệm = qua đó đưa ra hướng giải quyết
*/
