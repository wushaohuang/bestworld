
![Schneider Electric Story - Word Cloud 2022](https://github.com/wushaohuang/wsh.github.io/assets/99949764/6a14f235-b43b-4803-b98d-2a3885b731cb)

- 初始化ratio并获取前端的值

- openSOExcludeFilterList指open so用户**未**选中的部分

- 处理OPEN SO的筛选器
- 如果用户选择了OPEN SO, 那就说明未选择的部分需要减去

- 开始计算数据主体
- 获取前端值、排序、聚合等参数

- 初始化日期数组**dates**(数据列)（注意：dates是包含past_due字段的），并根据by day/week/month生成指定的表头（列名）存放于dates中

- 初始化？？？？数组weightList，根据dates的大小分为0-3（*2）、4-7（*1）、8-∞（*0.5）

- 获取前端的demandDueSelections，将其拆分为FUTURE_DUE和PAST_DUE两个数组，并将两个数组合并到selectedCategory中

- queryReport1Demand()（item为dates中的元素）
    1. 判断selectedCategory是否为空，如果不为空，判断FUTURE_DUE和PAST_DUE中元素是否存在于CATEGORY中，如果存在，字段名为"'${item}'_QTY" AS "${item}"，否则，字段名为0 AS "${item}"。其中item为dates中的元素
    2. 如果用户未选中OSO_NORMAL，则0 AS "TIPS_${item}_OSO_NORMAL",否则"'${item}'_OSO_NORMAL" AS "TIPS_${item}_OSO_NORMAL"
    3. OSO_CB、UD_MID_AGING、UD_CB、UD_NORMAL、UD_LONG_AGING同理（与OSO_NORMAL相同）
    4. ------------------------------最外层字段取完-----------------------------------
    5. 基于底表DEMAND_SUPPLY_DEMAND_V，根据PIVOT进行行转列
    ```sql
      SUM(QUANTITY ${valueColumnName}) QTY,
      SUM(OSO_NORMAL ${valueColumnName}) OSO_NORMAL,
      SUM(OSO_CB ${valueColumnName}) OSO_CB,
      SUM(UD_MID_AGING ${valueColumnName}) UD_MID_AGING,
      SUM(UD_CB ${valueColumnName}) UD_CB,
      SUM(UD_NORMAL ${valueColumnName}) UD_NORMAL,
      SUM(UD_LONG_AGING ${valueColumnName}) UD_LONG_AGING
      -- 拼接上dates中的日期
    ```
    6. 根据以上转换后的表和转换后的字段，进行查询得到demandList

- queryReport1Supply()
    1. 整体框架 -> 创建MA_BASE(被MA_COMMIT使用)， 创建MA_COMMIT、PO_BASE、SUPPLY、PO_COMMIT四张临时表，然后将这四张临时表拼接起来（union all），然后PIVOT行转列，最后得到supplyList
    2. MA_BASE -> 源于DEMAND_SUPPLY_MANUAL_PO_COMMIT，根据USER_ID筛选，并根据MATERIAL, PLANT_CODE, COMMIT_DATE聚合
    3. MA_COMMIT -> 基于MA_BASE，'MANUAL' AS CONFIRM_CAT, 'Supply_PO' AS SUPPLY_CATEGORY，其余字段基本上是直接获取
    4. PO_BASE -> 基于DEMAND_SUPPLY_MANUAL_PO_COMMIT，根据USER_ID筛选
    5. SUPPLY -> 基于DEMAND_SUPPLY_SUPPLY_V，并根据PURCH_ORDER_NUMBER, PURCH_ORDER_ITEM聚合
    6. PO_COMMIT -> 基于PO_BASE，join SUPPLY；'Supply_PO' AS SUPPLY_CATEGORY
    ```
    ------------------------------临时表整理完成-----------------------------------
    ```
    7. 接下来做拼接，拼接由DEMAND_SUPPLY_SUPPLY_V，MA_COMMIT ，PO_COMMIT 三部分组成
    8. 第一部分DEMAND_SUPPLY_SUPPLY_V ->
    ```sql
      DECODE(T.CONFIRM_CAT, 'FCST Commitment', 'Supply_' || T.DATA_LINE || '(FCST_Cmt)', 'Supply_' || T.DATA_LINE) CATEGORY, DECODE(T.CONFIRM_CAT, 'FCST Commitment', T.QUANTITY * ${fcstRatio}, T.QUANTITY) QUANTITY,
      -- 最核心where条件 -> 
      MATERIAL + PLANT_CODE不在MA_COMMIT中；PURCH_ORDER_NUMBER+PURCH_ORDER_ITEM不在PO_COMMIT中
      ```
    9. 第二部分MA_COMMIT ->
      ```sql
        'Supply_Manual' CATEGORY, DECODE(T.CONFIRM_CAT, 'FCST Commitment', T.QUANTITY * ${fcstRatio}, T.QUANTITY) QUANTITY,
      ```
    10. 第三部分
      ```sql
        'Supply_Manual' CATEGORY, DECODE(T.CONFIRM_CAT, 'FCST Commitment', T.QUANTITY * ${fcstRatio}, T.QUANTITY) QUANTITY
      ```
    // ------------------------------拼接完成-----------------------------------
    11. 根据PIVOT进行行转列
      ```sql
      SUM(QUANTITY${valueColumnName}) QTY,
                        SUM(AB${valueColumnName}) AB,
                        SUM(LA${valueColumnName}) LA,
                        SUM(NON${valueColumnName}) NON,
                      SUM(MANUAL_VALUE) MANUAL_VALUE
      -- 拼接上dates中的日期
      ```
    12. 根据以上转换后的表和转换后的字段，进行查询得到supplyList

- demandList
    1. 基于demandList将demand信息分为2类, 一类是有group的, 一类是没有group的
    2. 初始化demandGroupMap和demandMaterialMap
    	- 判断GROUP_MATERIAL字段是否为空，如果为空，则将获取到的“MATERIAL@PLANT_CODE”作为键值对的键存于demandMaterialMap中
    	- 如果不为空，则将获取到的“GROUP_MATERIAL@PLANT_CODE”作为键值对的键存于demandGroupMap中

    3. 对demandGroupMap和demandMaterialMap分别算Total, 放在两个新的Map里（demandGroupTotalMap和demandMaterialTotalMap）
    4. 初始化demandGroupTotalMap和demandMaterialTotalMap

    5. demand group
    	- 针对demandGroupMap中的元素，将它的元素们根据material+plant_code汇总
    	- 不是dates元素或者TIPS开头的字段，在多行数据的情况下只取一行数据；日期或者TIPS开头的字段，在多行数据的情况下该字段取汇总值（sum）
      			- getOrDefault(key, default)如果存在key, 则返回其对应的value, 否则返回给定的默认值
    	- 对于每一个元素将其CATEGORY设置为Demand Total
    6. 最终将demandGroupMap中所有元素根据material + plant_code聚合后的结果存入demandGroupTotalMap中

    7. demand material
    	- 针对demandMaterialMap中的元素，将它的元素们根据material+plant_code汇总
    	- 不是dates元素或者TIPS开头的字段，在多行数据的情况下只取一行数据；日期或者TIPS开头的字段，在多行数据的情况下该字段取汇总值（sum）
    		- getOrDefault(key, default)如果存在key, 则返回其对应的value, 否则返回给定的默认值
    	- 对于每一个元素将其GROUP_MATERIAL设置为""，CATEGORY设置为Demand Total
    8. 最终将demandMaterialMap中所有元素根据material + plant_code聚合后的结果存入demandMaterialTotalMap中

- supplyList
    1. 基于supplyList将supply信息分为2类, 一类是有group的, 一类是没有group的
    2. 初始化supplyGroupMap和supplyMaterialMap
    	- 判断GROUP_MATERIAL字段是否为空，如果为空，则将获取到的“MATERIAL@PLANT_CODE”作为键值对的键存于demandMaterialMap中
    	- 如果不为空，则将获取到的“GROUP_MATERIAL@PLANT_CODE”作为键值对的键存于demandGroupMap中
    3. 对supplyGroupMap和supplyMaterialMap分别算Total, 放在两个新的Map里（supplyGroupTotalMap和supplyMaterialTotalMap）
    4. 初始化supplyGroupTotalMap和supplyMaterialTotalMap
    5. supply group
    	- 针对supplyGroupMap中的元素，将它的元素们根据material+plant_code汇总
    	- 不是dates元素或者TIPS开头的字段，在多行数据的情况下只取一行数据；日期或者TIPS开头的字段，在多行数据的情况下该字段取汇总值（sum）
      		- getOrDefault(key, default)如果存在key, 则返回其对应的value, 否则返回给定的默认值
    	- 对于每一个元素将其CATEGORY设置为Demand Total
    6. 最终将supplyGroupMap中所有元素根据material + plant_code聚合后的结果存入supplyGroupTotalMap中
    7. supply material
    	- 针对supplyMaterialMap中的元素，将它的元素们根据material+plant_code汇总
    	- 不是dates元素或者TIPS开头的字段，在多行数据的情况下只取一行数据；日期或者TIPS开头的字段，在多行数据的情况下该字段取汇总值（sum）
      		- getOrDefault(key, default)如果存在key, 则返回其对应的value, 否则返回给定的默认值
    	- 对于每一个元素将其GROUP_MATERIAL设置为""，CATEGORY设置为Supply Total
    8. 最终将supplyMaterialMap中所有元素根据material + plant_code聚合后的结果存入supplyMaterialTotalMap中

- 查询comments
    1. 查询当前周数并存于weekNo中(queryCurrentWeek())
    2. queryReport1CommentsByUserid()
	    - 基于DEMAND_SUPPLY_COMMENTS
	    - where条件为周数weekNo和用户id
    3. 初始化commentsMap，并将commentsList中的MATERIAL@PLANT_CODE作为键，COMMENTS作为值存入commentsMap

- 获取Critical Level表达式
	1. queryReport1CriticalScript()
		- 在DEMAND_SUPPLY_CRITICAL_LEVEL_SETTINGS中获取Critical Level

- 计算 Balance
	- 初始化groupBalanceMap和materialBalanceMap和balance
	- 计算group balance
		1. queryReport1Group() 获取demand和supply中所有为group的料号信息（根据T.GROUP_MATERIAL IS NOT NULL进行筛选）
			- 基于DEMAND_SUPPLY_DEMAND_V和DEMAND_SUPPLY_SUPPLY_V根据material+plant_code进行聚合，最后将两张表的结果进行拼接
			- 注意：针对上边两张表的查询中有INNER JOIN DEMAND_SUPPLY_CRITICAL_MATERIAL_V on USER_ID = #{session.userid,jdbcType=VARCHAR}查询条件，故要注意只看用户自己的
			- WHERE T.GROUP_MATERIAL IS NOT NULL筛选出所有属于group的物料
			- 聚合字段
			```sql
				GROUP_MATERIAL AS "material",
				PLANT_CODE AS "plantCode",
				MAX(OPEN_PO_QTY) as "OPEN_PO_QTY",
				MAX(STOCK_ON_HAND) AS "currentStock",
				MAX(SAFETY_STOCK * ${ssRatio}) AS "safetyStock",
				MAX(AMF * ${amfRatio}) as "amf",
				MAX(AMU * ${amuRatio}) as "amu",
				MAX(MTD_ORDER_INTAKE) as "mtd_order_intake",
				MAX(MTD_SALES) as "mtd_sales"
			```
		2. 将queryReport1Group()查询结果赋值给demandGroupList
		3. 对demandGroupList中的material@plant_code进行遍历，对每一个元素，获取demandGroupTotalMap.getOrDefault(group.getKey(), new HashMap并赋值给demand
		4. 对demandGroupList中的material@plant_code进行遍历，对每一个元素，获取supplyGroupTotalMap.getOrDefault(group.getKey(), new HashMap并赋值给supply
		5. demand和group中存放的是每个物料对应的详细数据ex：{PAST_DUE: 2000; TIPS_20230511_OSO_NORMAL: 1000}
		- 计算非数据列
			1. 分别将demand和supply中的非数据列（不在dates中的列）放入balance中，且 balance.put("CATEGORY", "Balance");
		- 计算数据列
			1. 对于dates中每一个元素（对应表中的每一列）进行遍历
			2. 计算这个物料的可用库存
			3. 计算balance
				- By Day
					- supply is null
						- balance = 可用库存 - 需求
					- supply is not null
						- 如果当日供给不为空, 则加到明日
				- Others
					- supply is null
						- balance = 可用库存 - 需求
					- supply is not null
						- balance = 可用库存 - 需求 + 供给
				- 如果是week, 还要计算critical_level的变量
	- 计算material balance与group balance方式一模一样
	
- 初始化balanceWeightList
	- 将groupBalanceMap中的数据转换为指定数据格式，并存于balanceWeightList
		- 数据格式 [{key:"ZB5AT84@O001"; type:"GROUP", orderMap: {weight: -321996.25045}}]
	- 将materialBalanceMap中的数据转换为指定数据格式，并存于balanceWeightList
		- 数据格式 [{key:"ZB5AT84@O001"; type:"MATERIAL", orderMap: {weight: -321996.25045}}]

- DemandSupplyWeight中存放的是所有查询出来的基本结果，
	- key：material@plant_code
	- type: material/group
	- orderMap: 排序列{weight: -321996.25045}

- 排序，默认根据widght进行升序排序
	- DemandSupplyWeight 排序, 支持多列排序

- 计算Total Demand和Total Supply
	- 初始化amuTotal，amfTotal，ssTotal，sohTotal，opoTotal，moiTotal，msTotal，demandTotal，supplyTotal，balanceTotal

- 遍历balanceWeightList（此时包含页面中所有material@plant_code）
	- 如果是group
		1. 对于demandGroupTotalMap的每一个元素
			- 将它里面的数据列直接增量存于demandTotal中
			- ssTotal += SAFETY_STOCK
			- amuTotal += AMU
			- amfTotal += AMF
			- sohTotal += STOCK_ON_HAND
			- opoTotal += OPEN_PO_QTY
			- moiTotal += MTD_ORDER_INTAKE
			- msTotal += MTD_SALES
		2. 对于supplyGroupTotalMap的每一个元素
			- 将它里面的数据列直接增量存于supplyTotal中
			- 注意，此处有一个判断，对于相同的material+plant_code，如果在遍历demandGroupTotalMap时算过了ssTotal、amuTotal...等，在遍历supplyGroupTotalMap时就不会再重复追加了
				- ssTotal += SAFETY_STOCK
				- amuTotal += AMU
				- amfTotal += AMF
				- sohTotal += STOCK_ON_HAND
				- opoTotal += OPEN_PO_QTY
				- moiTotal += MTD_ORDER_INTAKE
				- msTotal += MTD_SALES

	- 如果是material
		1. 对于demandMaterialTotalMap的每一个元素
			- 将它里面的数据列直接增量存于demandTotal中
			- ssTotal += SAFETY_STOCK
			- amuTotal += AMU
			- amfTotal += AMF
			- sohTotal += STOCK_ON_HAND
			- opoTotal += OPEN_PO_QTY
			- moiTotal += MTD_ORDER_INTAKE
			- msTotal += MTD_SALES
		2. 对于supplyMaterialTotalMap的每一个元素
			- 将它里面的数据列直接增量存于supplyTotal中
			- 注意，此处有一个判断，对于相同的material+plant_code，如果在遍历demandGroupTotalMap时算过了ssTotal、amuTotal...等，在遍历supplyGroupTotalMap时就不会再重复追加了
				- ssTotal += SAFETY_STOCK
				- amuTotal += AMU
				- amfTotal += AMF
				- sohTotal += STOCK_ON_HAND
				- opoTotal += OPEN_PO_QTY
				- moiTotal += MTD_ORDER_INTAKE
				- msTotal += MTD_SALES

- 将上边算好的ssTotal、amfTotal、amuTotal、sohTotal、opoTotal、moiTotal、msTotal分别存放入（put）demandTotal、supplyTotal、balanceTotal中

- 计算Total Balance
	1. 这个物料的可用库存 = 当前库存 - SS - AMU - AMF
	2. 如果是By Day, 当日供给无法抵消当日需求
		- balance = 可用库存 - 需求
		- 如果当日供给不为空, 则加到明日
	3. 如果不是By Day
		- balance = 可用库存 - 需求 + 供给
	4. 最终将balance存入balanceTotal中

- 根据用户选择的显示行
	1. 由demandSupplySelections获取用户勾选上的rows

- 计算TOTAL PO_COVERAGE_DAYS =  (supplyopenPoQty + supplyStockOnHand) / (supplyAmf / 30)

- 将demandTotal、supplyTotal、balanceTotal分别存入resultList中

- 因为前面demandMaterialMap、demandMaterialTotalMap、supplyMaterialMap、supplyMaterialTotalMap、materialBalanceMap、demandGroupMap、demandGroupTotalMap、supplyGroupMap、supplyGroupTotalMap、groupBalanceMap中已经包含每颗料的基本信息，所以接下来只需要添加一些指定的计算逻辑即可，例如PO_COVERAGE_dAYS
- 针对每一个物料单独做处理，根据DEMAND_DETAILS、DEMAND_TOTAL、SUPPLY_DETAILS、SUPPLY_TOTAL、BALANCE分别生成单独的行供前端使用
	- is group?
		1. DEMAND_DETAILS
			- 如果demandGroupMap存在这个料（material@plant_code）
				- PO_COVERAGE_DAYS = (openPoQty + stockOnHand) / (AMF / 30)
					- openPoQty = OPEN_PO_QTY
					- AMF = AMF
					- stockOnHand = STOCK_ON_HAND
			- 否则
				- CATEGORY = Demand_No Records
				- COMMENTS = null
				- CRITICAL_LEVEL = null
				- 其他dates中的数据列 = null
		3. DEMAND_TOTAL
			- 如果demandGroupTotalMap存在这个料（material@plant_code）
				- PO_COVERAGE_DAYS = (openPoQty + stockOnHand) / (AMF / 30)
					- openPoQty = OPEN_PO_QTY
					- AMF = AMF
					- stockOnHand = STOCK_ON_HAND
			- 否则
				- CATEGORY = Demand Total
				- COMMENTS = null
				- CRITICAL_LEVEL = null
				- 其他dates中的数据列 = null
		5. SUPPLY_DETAILS
			- 如果supplyGroupMap存在这个料（material@plant_code）
				- PO_COVERAGE_DAYS = (openPoQty + stockOnHand) / (AMF / 30)
					- openPoQty = OPEN_PO_QTY
					- AMF = AMF
					- stockOnHand = STOCK_ON_HAND
			- 否则
				- CATEGORY = Supply_No Records
				- COMMENTS = null
				- CRITICAL_LEVEL = null
				- 其他dates中的数据列 = null
		7. SUPPLY_TOTAL
			- 如果supplyGroupTotalMap存在这个料（material@plant_code）
				- PO_COVERAGE_DAYS = (openPoQty + stockOnHand) / (AMF / 30)
					- openPoQty = OPEN_PO_QTY
					- AMF = AMF
					- stockOnHand = STOCK_ON_HAND
			- 否则
				- CATEGORY = Supply Total
				- COMMENTS = null
				- CRITICAL_LEVEL = null
				- 其他dates中的数据列 = null
		9. BALANCE
			- PO_COVERAGE_DAYS = (openPoQty + stockOnHand) / (AMF / 30)
				- openPoQty = OPEN_PO_QTY
				- AMF = AMF
				- stockOnHand = STOCK_ON_HAND
	- is not group? （和上边isGroup同理）
		1. DEMAND_DETAILS
		2. DEMAND_TOTAL
		3. SUPPLY_DETAILS
		4. SUPPLY_TOTAL
		5. BALANCE


		





