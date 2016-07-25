## lua编程策略简单示例

### 简单均线交叉

以下是简单的均线交叉策略（因为不使用遗传算法，因此“可以使用的公式值的数量”填0）：

```lua
--[[ This is the example codes for the moving average crossover strategy
lua策略简单样例，该策略是均线交叉策略，5日均线上传10日均线则全仓买入，5日(短期)
均线下穿10日(长期)均线则全仓卖出
无止损
无遗传算法优化
--]]
return function ()
    local pv = portfolio.portfolioValue() -- 股票账户价值
    print("pv:",pv,"cash:",portfolio.cash()) -- 打印股票账户价值到策略输出日志
    local eachTargetMoney = pv/#universe -- 每只股票分得的操作资金
    print("eachTargetMoney:",eachTargetMoney) -- 打印每只股票分得的操作资金到策略输出日志
    for target, stat in pairs(bar()) do  --for循环遍历每只股票
        local po = portfolio.position(target) -- 股票持仓状态
        local MAS = stat['MA5'] -- 5日均线(短期)，续在“高阶指标数据项”内添加5日均线
        local MAL = stat['MA10'] -- 10日均线(长期)，续在“高阶指标数据项”内添加10日均线
        local LMAS = stat['LMA5'] -- 前一日5日均线值，续在“高阶指标数据项”用ref函数添加前一日5日均线
        local LMAL = stat['LMA10'] -- 前一日10日均线值，续在“高阶指标数据项”用ref函数添加前一日10日均线
        local price = stat['CLOSE'] -- 当前股价

        -- 以下是策略逻辑
        local want = (function()
            if MAS > MAL and LMAS < LMAL and po.quantity == 0 then
                return eachTargetMoney/price -- 触发买入
            end
            if MAS < MAL and LMAS > LMAL and po.quantity > 0 then
                return 0 -- 触发卖出
            end
            return po.quantity -- 保持持仓
        end)()

        if po.quantity ~= want then
            ret = orderShares(target, want-po.quantity) -- 下单
            --print("order :",ret,",",want,",",po)
        end
    end
end
```

高阶指标数据项应该填入以下内容：

```
MA5   : MA(5,CLOSE);
MA10  : MA(10,CLOSE);
LMA5  : REF(1,MA5);
LMA10 : REF(1,MA10);
```

高阶指标数据的填入方式为：

点击 "高阶指标数据项" -&gt; 点击 "使用指标编辑模式" -&gt; 修改 "定义指标公式"。

关于高阶指标的帮助，将在近期公布，目前高阶指标中函数的参数设定与一般使用的股票软件工具不相同。将来我们将会将其调整到一致，相应地，用户需要对自己正在使用的指标进行一些调整。

### 简单均线交叉加止损

如果需要将以上策略加入止损比率，则代码应该如下（因为不使用遗传算法，因此“可以使用的公式值的数量”填0）：

```lua
--[[ This is the example codes for the moving average crossover strategy
lua策略简单样例，该策略是均线交叉策略，5日均线上传10日均线则全仓买入，5日
均线下穿10日均线则全仓卖出
有止损，止损设置在20%
无遗传算法优化
--]]
local drawdownrate = 0.20 --定义止损比率
local hightable = {} -- 定义最高价table
return function ()
    local pv = portfolio.portfolioValue() -- 股票账户价值
    print("pv:",pv,"cash:",portfolio.cash()) -- 打印股票账户价值到策略输出日志
    local eachTargetMoney = pv/#universe -- 每只股票分得的操作资金
    print("eachTargetMoney:",eachTargetMoney) -- 打印每只股票分得的操作资金到策略输出日志
    for target, stat in pairs(bar()) do -- for循环遍历每只股票
        local po = portfolio.position(target) -- 股票持仓状态
        local high = stat['HIGH'] -- 最高价，用于止损的计算
        local MAS = stat['MA5'] -- 5日均线，续在“高阶指标数据项”内添加5日均线
        local MAL = stat['MA10'] -- 10日均线，续在“高阶指标数据项”内添加10日均线
        local LMAS = stat['LMA5'] -- 前一日5日均线值，续在“高阶指标数据项”用ref函数添加前一日5日均线
        local LMAL = stat['LMA10'] -- 前一日10日均线值，续在“高阶指标数据项”用ref函数添加前一日10日均线
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
            if MAS > MAL and LMAS < LMAL and po.quantity == 0 then
                return eachTargetMoney/price -- 触发买入
            end
            if MAS < MAL and LMAS > LMAL and po.quantity > 0 then
                return 0 -- 触发卖出
            end
            return po.quantity
        end)()

        if po.quantity ~= want then
            ret = orderShares(target, want-po.quantity) -- 下单
            --print("order :",ret,",",want,",",po)
        end
    end
end
```

高阶指标数据项应该填入以下内容：

```
MA5   : MA(5,CLOSE);
MA10  : MA(10,CLOSE);
LMA5  : REF(1,MA5);
LMA10 : REF(1,MA10);
```

### 简单均线交叉加止损，并且加入遗传算法优化

如果需要将20日均线利用遗传算法进行优化成某表达式，并将该表达式和90日均线进行金叉死叉策略，则可以使用以下代码（因为需要使用遗传算法优化一个表达式，因此“可以使用的公式值的数量”填1）：

```lua
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
    print("pv:",pv,"cash:",portfolio.cash()) -- 打印股票账户价值到策略输出日志
    local eachTargetMoney = pv/#universe -- 每只股票分得的操作资金
    print("eachTargetMoney:",eachTargetMoney) -- 打印每只股票分得的操作资金到策略输出日志
    for target, stat in pairs(bar()) do -- for循环遍历每只股票
        local po = portfolio.position(target) -- 股票持仓状态
        local high = stat['HIGH'] -- 最高价，用于止损计算
        local MAS= stat['MA5'] -- 5日均线，续在“高阶指标数据项”内添加20日均线
        --local MAL = stat['MA10'] -- 10日均线，续在“高阶指标数据项”内添加90日均线
        local LMAS = stat['LMA5']
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
                and formula1 > MAS and lformula1 < LMAS and po.quantity == 0
        local conditionSell = formula1 ~= nil and lformula1 ~= nil
                and formula1 < MAS and lformula1 > LMAS and po.quantity > 0

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
            ret = orderShares(target, want-po.quantity) -- 下单
            --print("order :",ret,",",want,",",po)
        end
        lastformula1[target] = formula1 -- 记录公式1值，提供给下一周期的计算
    end
end
```

高阶指标数据项应该填写如下：

```
MA5   : MA(5,CLOSE);
MA10  : MA(10,CLOSE);
LMA5  : REF(1,MA5);
LMA10 : REF(1,MA10);
```

### 排序买入策略

```lua
local num = 5
local adratio = 0.1
local baratio = 0.05
return function ()
    local formulasort = {}
    local winnertable = {}
    local holdtable = {}
    local pv = portfolio.portfolioValue()
    local targetvalue = pv/num
    print("pv:",pv,"cash:",portfolio.cash(),"现金占比： ",portfolio.cash()/pv)
    local eachTargetUplimit = pv/num*(1+adratio)
    local cash = portfolio.cash()

    local function intable(item, table)
        for k,v in pairs(table) do
            if item == v then
                return true
            end
        end
        return false
    end

    for target, stat in pairs(bar()) do
        local po = portfolio.position(target)
        local formula1 = formula(target,0)
        local price = stat['CLOSE']
        --print(target,price)
        table.insert(formulasort,{symbol = target, val = formula1})
        if po.quantity >0 then
            print("now i have :", target)
            table.insert(holdtable,target)
        end
    end
    table.sort(formulasort,function(a,b) return a.val>b.val end )

    --获取winnertable
    for i=1, num do
        print(formulasort[i].symbol, formulasort[i].val)
        table.insert (winnertable, formulasort[i].symbol)
    end
    --获取winnertable结束

    --卖出操作
    for k,target in pairs(holdtable) do
        print("空table不会运行")
        if intable(target,winnertable) then
            print("先不卖出", target)
        else
            print("卖出", target)
            ret = orderShares(target, -portfolio.position(target).sellable)
        end

        print("卖出操作后，现金： " ,portfolio.cash(),"现金占比： ",portfolio.cash()/pv)
        --adjust ratio
        print(portfolio.position(target).value,"----",eachTargetUplimit)
        if portfolio.position(target).value > eachTargetUplimit then
            ret = orderValue(target, math.floor(pv*(1+baratio)/num-portfolio.position(target).value ))
            print("调整仓位到适当水平：" ,target, ret)
        end
        print("调整操作后，现金： " ,portfolio.cash(),"现金占比： ",portfolio.cash()/pv)
    end
    --卖出操作结束

    --买入操作
    local buytable = {}
    local sbuytable = {}
    for k,target in pairs(winnertable) do
        --如果在adratio上下范围内，那就不调整。否则调整到pv/num水平附近
        --只考虑买入方向
        rtargetvalue = portfolio.position(target).value

        if rtargetvalue < targetvalue*(1+adratio) and rtargetvalue > targetvalue*(1-adratio) then
            print("不买入")
        else
            if rtargetvalue < targetvalue*(1-adratio) then
                if portfolio.position(target).value == 0 then
                    table.insert(buytable,target)
                else
                    table.insert(sbuytable,target)
                end
            end
        end

    end
    cash = portfolio.cash()
    for k,target in pairs(buytable) do
        ret = orderValue(target, cash/#buytable )
        print("买入操作：" ,target, ret)
    end
    cash = portfolio.cash()
    for k,target in pairs(sbuytable) do
        ret = orderValue(target, cash/#sbuytable )
        print("买入操作：" ,target, ret)
    end

    print("买入操作后，现金： " ,portfolio.cash(),"现金占比： ",portfolio.cash()/pv)
    --买入操作结束
end
```

### 趋势跟踪策略

```lua
local drawdownrate = 0.05
local lastformula1 = {}
local lastformula2 = {}
local hightable = {}
return function ()
    local pv = portfolio.portfolioValue()
    print("pv:",pv,"cash:",portfolio.cash())
    local eachTargetMoney = pv/#universe
    print("eachTargetMoney:",eachTargetMoney)
    for target, stat in pairs(bar()) do
        local po = portfolio.position(target)

        local MA5 = stat['MA5']

        local price = stat['CLOSE']
        local high = stat['HIGH']
        local formula1 = formula(target,0)
        local formula2 = formula(target,1)


        local lformula1 = lastformula1[target]
        local lformula2 = lastformula2[target]

        -- 同步hightable
        if po.quantity > 0 then
            if hightable[target] == nil or hightable[target]<high then
                hightable[target] = high
            end
        else
            hightable[target] = nil
        end

        local conditionA = formula1 ~= nil and formula1 > MA5
        local conditionB = formula1 ~= nil and formula1 <= MA5
        local conditiondrawback = hightable[target] ~= nil and price<(1-drawdownrate)*hightable[target]

        print(target, ' his-high is ',hightable[target])
        -- 结束同步hightable

        local want = (function()
            if conditiondrawback then
                return 0
            end
            if conditionA and po.quantity > 0 and po.quantity*price/eachTargetMoney<=ratio*1.05 then
                return eachTargetMoney/price
            end
            if conditionA and po.quantity == 0 then
                return (eachTargetMoney*ratio)/price
            end
            if conditionB and po.quantity > 0 then
                return 0
            end
            if conditionB and po.quantity == 0 then
                return po.quantity
            end
            return po.quantity
        end)()

        print(target.." po.quantity before transaction :  "..po.quantity.."sellable is : "..po.sellable)
        print("target :   ".. target .. "want :   ".. want.."position:   ".. po.quantity .."sellable is :   ".. po.sellable)
        print("target :   ", target , "condition buy :   ", conditionA , "condition sell :   " , conditionB, "conditiondrawback : " ,conditiondrawback )
        print("#### before target :   ", target , "个股仓位比例 :   ", po.quantity*price/eachTargetMoney)
        if po.quantity ~= want then
            ret = orderShares(target, math.floor((want-po.quantity)/100)*100)
            --print("order :",ret,",",want,",",po)
        end
        po = portfolio.position(target)
        print(target ," po.quantity after transaction : ",target,po.quantity,"sellable is : ", po.sellable)
        print("#### after target :   ", target , "个股仓位比例 :   ", po.quantity*price/eachTargetMoney)
        lastformula1[target] = formula1
        lastformula2[target] = formula2

    end
end
```


高阶指标数据项应该<span style="color:red">至少</span>包含，包含其他指标方便训练出出色的策略。

```
MA5   : MA(5,CLOSE);
```
