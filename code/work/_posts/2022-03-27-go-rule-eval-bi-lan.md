---
layout: post
title: 资损监控系统搭建（一）：Golang规则引擎调研
description: >
    关于资损监控系统的思考

image: 
  path: /assets/img/blog/example-content-ii.jpg
  srcset:
    1060w: /assets/img/blog/example-content-ii.jpg
    530w:  /assets/img/blog/example-content-ii@0,5x.jpg
    265w:  /assets/img/blog/example-content-ii@0,25x.jpg
sitemap: true
comments: true
---

资损问题应该是每个商业系统的头等大事，如何发现资损和止血止损的速度是衡量业务团队的重要指标，
本文讲述我对于搭建资损监控系统的思考。

整个系列应该会分为五章吧：
    - 规则引擎调研: 为系统引入实时性，快速迭代的功能
    - 基于消息的实时资损监控系统设计：简单低廉的内部方案，快速发现并及时止损，有优有劣。
    - 小时级对账系统：牺牲一部分实时性，从而实现跨端校验。
    - 上下游资金离线核对系统设计：业务代码无入侵，构建成本高，但是可靠性与准确性强！实时对账了属于是。
    - 代码重构：一个资损问题反映出来的，是千百个业务逻辑的耦合。
    - 
本文是第一章，调研了一下Go生态里的一些开源规则引擎第三方库，多方对比后，最终采用的是B站开源的(gengine)[https://github.com/bilibili/gengine]

## 为什么需要规则引擎

业务早期，开发们将大量硬编码的规则逻辑混合在业务逻辑中实现。

但是随着业务逻辑增加，场景增多，业务规则的硬编码方式开始捉荆见肘：
    - 策略分散无法管理
    - 逻辑同业务强耦合
    - 策略更新迭代率受限于开发，对接成本高

此时，可以通过接入一套可配置化的规则管理平台，进而实现业务规则与业务逻辑解耦、降低试错成本，提高迭代速度。

规则引擎使用场景：
    - 规则逻辑：规则变更期望脱离于开发人员，脱离coding
    - 旁路场景：如风控，对账，旁路监控
    - 安全校验：需要快速做出响应和决策
    - 其他需要流程编排的场景

## 规则引擎与配置中心区别

以下是个人见解：

- 配置中心是系统参数层，规则引擎是业务规则层
- 配置中心的“规则”是单维度，静态的，比如通过配置支持退款的category 列表来判断订单的category是否在其中，从而判断是否支持退款。

但是在实际业务场景中，判断一个订单是否支持退款还需要涉及订单的订单状态，是否有退款单等等状态。需要多维度的规则判断。

在后续的实践中，我把规则引擎的规则放在了配置中心上进行管理和更新。

## 几个开源的规则引擎库

|  开源库  |  语法 | 性能 ｜ 文档 ｜ 社区｜ 扩展性 ｜ 流程编排 ｜ 适用场景｜
| :----   |:----  |:----|:----|:----|:---- |:----  |
| go-valuate  | 仅支持逻辑表达式 |中｜英文/少｜不维护｜弱/不支持传入结构体｜否｜逻辑表达式，算数表达式的场景｜
| expr  | 仅支持逻辑表达式 |低｜英文/少｜一般｜弱/不支持传入结构体｜否｜逻辑表达式，算数表达式的场景｜
| gengine  | 类似DRL，支持逻辑分支 |高｜中文/多｜一般/B站开源，社群｜强/支持传入结构体与方法｜是｜复杂、易变的业务规则｜

最终我们选择的是`gengine`，开源文档写的挺详细的，[可以看看](https://github.com/bilibili/gengine/wiki/)。

## 用法&性能对比

性能对比代码都放在[这个仓库](https://github.com/Jun10ng/go-rule-eng-research)了，自取。


### go-valuate
不支持传入结构体到表达式当中。只能把用的字段拆成基本数据结构传入
逻辑表达式

```
func main()  {
    req := define.CreateOrderRequest{
        TotalAmount: 1,
        FinalPrice: 0,
        AdminFee: -1,
    }
    exp1, _ := govaluate.NewEvaluableExpression("TotalAmount< 0"); //
    parameters := make(map[string]interface{}, 8)
    parameters["TotalAmount"] = req.TotalAmount; 
 
    /*
        不支持
        parameters["req"] = req
        exp1, _ := govaluate.NewEvaluableExpression("req.TotalAmount< 0");
    */
 
 
    result, _ := exp1.Evaluate(parameters);
    if result.(bool){
        fmt.Println("exp1 err")
    }
}
```

### expr
支持传入结构体与方法
但是仅支持逻辑表达式，通过 && || 链接
```
const rule = `
       req.TotalAmount>0
    && req.FinalPrice>0
    && req.AdminFee>0`
func main() {
    req := &define.CreateOrderRequest{
        TotalAmount: 1,
        FinalPrice: 0,
        AdminFee: -1,
    }
    env := map[string]interface{}{
        "req":   req,
        "displayTotal": displayTotal, 
    }
 
 
     
    program, err := expr.Compile(rule, expr.Env(env))
    if err != nil {
        panic(err)
    }
 
    output, err := expr.Run(program, env)
    if err != nil {
        panic(err)
    }
 
    fmt.Println(output)
}
```

### gengine
```
// DRL 规则声明
const rule1 = `
rule "TotalAmount"
begin
        displayTotal(req.GetTotalAmount())
        if req.GetTotalAmount() < 0{
            return false
        }
        return true
end
`
 
 
 
func displayTotal(total int64)string{
    return fmt.Sprintf("total is %v \n",total)
}
func main()  {
    req := &define.CreateOrderRequest{
        TotalAmount: 1,
        FinalPrice: 0,
        AdminFee: -1,
    }
    dataContext := context.NewDataContext()
    //注入初始化的结构体
    dataContext.Add("req", req)
    // 注入函数
    dataContext.Add("displayTotal",displayTotal)
    ruleBuilder := builder.NewRuleBuilder(dataContext)
    err := ruleBuilder.BuildRuleFromString(rule1)
    if err != nil{
        fmt.Println("err:%s ", err)
    }else{
        eng := engine.NewGengine()
        _ = eng.Execute(ruleBuilder,true)
        result,_ := eng.GetRulesResultMap()
        r := result["TotalAmount"]
        if r == nil{
            fmt.Println("r is nil")
        }else {
            fmt.Printf("Total amount check result is %v\n",r.(bool))
        }
 
    }
}
```



### benchmark 测试结果

```
goos: darwin
goarch: amd64
pkg: ruleng
cpu: Intel(R) Core(TM) i9-9880H CPU @ 2.30GHz
Benchmark_govaluate
Benchmark_govaluate-16       3840882           267.2 ns/op
Benchmark_expr
Benchmark_expr-16            8106598           128.2 ns/op
Benchmark_gengine
Benchmark_gengine-16         1918964           665.3 ns/op
PASS
```

