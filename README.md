
![Schneider Electric Story - Word Cloud 2022](https://github.com/wushaohuang/wsh.github.io/assets/99949764/6a14f235-b43b-4803-b98d-2a3885b731cb)

- 初始化ratio并获取前端的值

- openSOExcludeFilterList指open so用户**未**选中的部分

- 处理OPEN SO的筛选器
- 如果用户选择了OPEN SO, 那就说明未选择的部分需要减去

- 开始计算数据主体
- 获取前端值、排序、聚合等参数

- 初始化日期数组dates（注意：dates是包含past_due字段的），并根据by day/week/month生成指定的表头（列名）存放于dates中

- 初始化？？？？数组weightList，根据dates的大小分为0-3（*2）、4-7（*1）、8-∞（*0.5）

- 获取前端的demandDueSelections，将其拆分为FUTURE_DUE和PAST_DUE两个数组，并将两个数组合并到selectedCategory中

- queryReport1Demand（item为dates中的元素）
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

- queryReport1Supply
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
    1. 查询当前周数并存于weekNo中(queryCurrentWeek)
    2. queryReport1CommentsByUserid
	    - 基于DEMAND_SUPPLY_COMMENTS
	    - where条件为周数weekNo和用户id
    3. 初始化commentsMap，并将commentsList中的MATERIAL@PLANT_CODE作为键，COMMENTS作为值存入commentsMap







