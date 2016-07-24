## lua编程策略简单示例

### 简单均线交叉

以下是简单的均线交叉策略（因为不使用遗传算法，因此“可以使用的公式值的数量”填0）：


```lua
--[[ This is the example codes for the moving average crossover strategy
lua策略简单样例，该策略是均线交叉策略，20日均线上传90日均线则全仓买入，20日
均线下穿90日均线则全仓卖出
无止损
无遗传算法优化
--]]
return function ()
    local pv = portfolio.portfolioValue() -- 股票账户价值
    print("pv:",pv,"cash:",portfolio.cash()) -- 打印股票账户价值到屏幕
    local eachTargetMoney = pv/#universe -- 每只股票分得的操作资金
    print("eachTargetMoney:",eachTargetMoney) -- 打印每只股票分得的操作资金到屏幕
    for target, stat in pairs(bar()) do -- for循环遍历每只股票
          local po = portfolio.position(target) -- 股票持仓状态

          local MA20 = stat['MA20'] -- 20日均线，续在“高阶指标数据项”内添加20日均线
          local MA90 = stat['MA90'] -- 90日均线，续在“高阶指标数据项”内添加90日均线
          local LMA20 = stat['LMA20'] -- 前一日20日均线值，续在“高阶指标数据项”用ref函数添加前一日20日均线
          local LMA90 = stat['LMA90'] -- 前一日90日均线值，续在“高阶指标数据项”用ref函数添加前一日90日均线
          local price = stat['CLOSE'] -- 当前股价

          -- 以下是策略逻辑
          local want = (function()
            if MA20 > MA90 and LMA20 < LMA90 and po.quantity == 0 then
              return eachTargetMoney/price -- 触发买入
            end
            if MA20 < MA90 and LMA20 > LMA90 and po.quantity > 0 then
              return 0 -- 触发卖出
            end
            return po.quantity
          end)()
          
          if po.quantity ~= want then
              ret = orderShares(target, math.floor((want-po.quantity)/100)*100) -- 下单
              --print("order :",ret,",",want,",",po)
          end
      end
  end
```  

高阶指标数据项应该填入以下内容：

```
MA20 : MA(20,CLOSE);
MA90 : MA(90,CLOSE);
LMA20 : REF(1,MA20);
LMA90 : REF(1,MA90);

```
### 简单均线交叉加止损

如果需要将以上策略加入止损比率，则代码应该如下（因为不使用遗传算法，因此“可以使用的公式值的数量”填0）：

```lua
--[[ This is the example codes for the moving average crossover strategy
lua策略简单样例，该策略是均线交叉策略，20日均线上传90日均线则全仓买入，20日
均线下穿90日均线则全仓卖出
有止损，止损设置在5%
无遗传算法优化
--]]
local drawdownrate = 0.05 --定义止损比率
local hightable = {} -- 定义最高价table
return function ()
    local pv = portfolio.portfolioValue() -- 股票账户价值
    print("pv:",pv,"cash:",portfolio.cash()) -- 打印股票账户价值到屏幕
    local eachTargetMoney = pv/#universe -- 每只股票分得的操作资金
    print("eachTargetMoney:",eachTargetMoney) -- 打印每只股票分得的操作资金到屏幕
    for target, stat in pairs(bar()) do -- for循环遍历每只股票
          local po = portfolio.position(target) -- 股票持仓状态

          local high = stat['HIGH'] -- 最高价，用于止损的计算
          local MA20 = stat['MA20'] -- 20日均线，续在“高阶指标数据项”内添加20日均线
          local MA90 = stat['MA90'] -- 90日均线，续在“高阶指标数据项”内添加90日均线
          local LMA20 = stat['LMA20'] -- 前一日20日均线值，续在“高阶指标数据项”用ref函数添加前一日20日均线
          local LMA90 = stat['LMA90'] -- 前一日90日均线值，续在“高阶指标数据项”用ref函数添加前一日90日均线
          local price = stat['CLOSE'] -- 当前股价

          -- 同步hightable
          if po.quantity > 0 then
              if hightable[target] == nil or hightable[target]<high then
                  hightable[target] = high
              end
          else
              hightable[target] = nil 
          end

          -- 以下是策略逻辑
          local want = (function()
            if hightable[target] ~= nil and price<(1-drawdownrate)*hightable[target] then
              return 0 -- 触发止损
            end  
            if MA20 > MA90 and LMA20 < LMA90 and po.quantity == 0 then
              return eachTargetMoney/price -- 触发买入
            end
            if MA20 < MA90 and LMA20 > LMA90 and po.quantity > 0 then
              return 0 -- 触发卖出
            end
            return po.quantity
          end)()
          
          if po.quantity ~= want then
              ret = orderShares(target, math.floor((want-po.quantity)/100)*100) -- 下单
              --print("order :",ret,",",want,",",po)
          end
      end
  end
```

高阶指标数据项应该填入以下内容：

```
MA20 : MA(20,CLOSE);
MA90 : MA(90,CLOSE);
LMA20 : REF(1,MA20);
LMA90 : REF(1,MA90);

```
### 简单均线交叉加止损，并且加入遗传算法优化

如果需要将20日均线利用遗传算法进行优化成某表达式，并将该表达式和90日均线进行金叉死叉策略，则可以使用以下代码（因为需要使用遗传算法优化两个表达式，因此“可以使用的公式值的数量”填1）：

```
--[[ This is the example codes for the moving average crossover strategy with genetic programming
lua策略简单样例，该策略的模型框架依然是均线交叉策略，但是具体哪些表达式构成金叉死叉
则是交给遗传算法训练出来
有止损，止损设置在5%
使用遗传算法优化
--]]
local drawdownrate = 0.05 --定义止损比率
local hightable = {} -- 定义最高价table
local lastformula1 = {} -- 定义公式1上一期table
return function ()
    local pv = portfolio.portfolioValue() -- 股票账户价值
    print("pv:",pv,"cash:",portfolio.cash()) -- 打印股票账户价值到屏幕
    local eachTargetMoney = pv/#universe -- 每只股票分得的操作资金
    print("eachTargetMoney:",eachTargetMoney) -- 打印每只股票分得的操作资金到屏幕
    for target, stat in pairs(bar()) do -- for循环遍历每只股票
          local po = portfolio.position(target) -- 股票持仓状态

          local high = stat['HIGH'] -- 最高价，用于止损计算
          local MA20 = stat['MA20'] -- 20日均线，续在“高阶指标数据项”内添加20日均线
          local MA90 = stat['MA90'] -- 90日均线，续在“高阶指标数据项”内添加90日均线
          local LMA90 = stat['LMA90']
          local price = stat['CLOSE'] -- 当前股价
          local formula1 = formula(target,0) -- 训练表达式1，第二个参数是计数，lua语言从0开始计数
          print("formula1:",formula1)
          local lformula1 = lastformula1[target] -- 昨日表达式1的值
          print("lformula1:",lformula1)

          -- 同步hightable，用于止损计算
          if po.quantity > 0 then
              if hightable[target] == nil or hightable[target]<high then
                  hightable[target] = high
              end
          else
              hightable[target] = nil 
          end

          -- 以下是策略逻辑
          local conditionStopLoss = hightable[target] ~= nil and price<(1-drawdownrate)*hightable[target]
          local conditionBuy = formula1 ~= nil and lformula1 ~= nil
                and formula1 > MA90 and lformula1 < LMA90 and po.quantity == 0
          local conditionSell = formula1 ~= nil and lformula1 ~= nil
                and formula1 < MA90 and lformula1 > LMA90 and po.quantity > 0

          local want = (function()
            if conditionStopLoss then
              return 0 -- 触发止损
            end
            if conditionBuy then
              return eachTargetMoney/price -- 触发买入
            end
            if conditionSell then
              return 0 -- 触发卖出
            end
            return po.quantity
          end)()

          if po.quantity ~= want then
              ret = orderShares(target, math.floor((want-po.quantity)/100)*100) -- 下单
              --print("order :",ret,",",want,",",po)
          end
          lastformula1[target] = formula1 -- 记录公式1值
      end
  end
```

高阶指标数据项应该填写如下：

```
MA20 : MA(20,CLOSE);
MA90 : MA(90,CLOSE);
LMA90 : REF(1,MA90);
```

