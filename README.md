# 饿了么订单网页插件

![image](https://cloud.githubusercontent.com/assets/1503156/22875717/2e1da55e-f208-11e6-83cf-5229ed1fd017.png)

面向桌面端 Chrome 和 Safari 的饿了么订单网页插件…… 帮你算出每个人要付多少，能统计餐盒、红包等费用。<br>
图中的红框部分是插件生成的，包含每用户的详单，还有最后的小结。（小结部分可手工复制或截图，方便发起者转发给其它人）

## 插件安装方法

参考 https://jareguo.github.io/ele-me-web-extend/ 中的方式进行，或者手工注册：<br>
在浏览器收藏夹中新建一个网址，将网址设为下面的完整代码。然后打开任一订单页面（需已下单），接着直接在收藏夹点击刚刚收藏的网址即可。

```js
javascript:(function () {
    var Yuan = '¥';
    var Credit = '当前插件版本 1.1.5';
    var rootEl, customerEls, otherFee, totalEl;
    var feePerCustomer, rate;
    var customers = [];

    function err (text) {
        text = '插件错误: ' + text;
        alert(text);
        return new Error(text);
    }

    function appendFooter () {
        var footer = document.createElement('span');
        footer.innerHTML = '<a href="https://jareguo.github.io/ele-me-web-extend/" target="_blank">' + Credit + '</a>';
        document.body.appendChild(footer);
    }

    function getEls () {
        rootEl = document.body.querySelector('div.spell-items');
        if (rootEl.childElementCount === 0) {
            throw err('无法解析内容，因为订单已完成');
        }
        customerEls = rootEl.querySelectorAll('section');
        if (customerEls.length === 0) {
            throw err('找不到用户元素');
        }
        otherFee = rootEl.querySelector('section.spell-otherfee');
        if (otherFee) {
            customerEls = Array.prototype.slice.call(customerEls, 0, -1);
        }
        else {
            throw err('找不到"其它费用"所在元素');
        }
        totalEl = rootEl.querySelector('div.spell-total > span');
        if (!totalEl) {
            /* <div class="spell-total">
               5人，32份商品<span>合计 ¥83.05</span></div> */
            throw err('找不到"合计"所在元素，订单可能已完成');
        }

    }
    function getCostEls (el) {
        return el.querySelectorAll('li>:first-child>:last-child');
    }
    function addEntry (parent, template, title, unit, cost) {
        var newEl = template.cloneNode(true);
        var items;
        items = newEl.querySelectorAll('span');
        if (items.length < 3) {
            throw err('找不到单元格');
        }
        items[0].innerHTML = title;
        items[1].innerText = unit;
        items[2].innerText = Yuan + cost;
        items[0].style.cssText = items[2].style.cssText =
            'user-select: text; -webkit-user-select: text;';
        items[1].style.width = '3rem';
        items[1].style.display = 'inline';
        parent.appendChild(newEl);
    }
    function getTitleEl (el) {
        var spans = el.querySelectorAll('header span');
        return spans[spans.length - 1];
    }
    function getCost (el) {
        var text = el.innerText.split(Yuan).splice(-1)[0];
        var cost = parseFloat(text);
        if (isNaN(cost)) {
            throw err('获取金额失败');
        }
        if (el.innerText[0] === '-') {
            cost = -cost;
        }
        return cost;
    }

    function sumFee (el) {
        var fee = 0, discount = 0;
        var els = getCostEls(el);
        for (var i = 0; i < els.length; i++) {
            var el = els[i];
            var cost = getCost(el);
            if (cost >= 0) {
                fee += cost;
            }
            else {
                discount += Math.abs(cost);
            }
        }
        return { fee, discount };
    }
    function computeTotal () {
        var { fee, discount } = sumFee(otherFee);
        /* 1. 实付 / (实付 + 优惠)，算出折扣 */
        var total = getCost(totalEl);
        rate = total / (total + discount);
        /* 2. 把其它支出费用均摊到每用户 */
        feePerCustomer = fee / customerEls.length;
    }

    function parseCustomers () {
        /* 3. 每用户应付 x 折扣，算出每用户实付 */
        customerEls.forEach(x => {
            var { fee, discount } = sumFee(x);
            var cost = fee - discount + feePerCustomer;
            var discountedCost = cost * rate;
            console.log(discountedCost);
            var newUl = x.querySelector('ul').cloneNode('true');
            newUl.innerHTML = '';
            var liTemplate = x.querySelector('li').cloneNode('true');
                var liChildren = liTemplate.children;
                for (var i = liChildren.length - 1; i >= 1; i--) {
                    liChildren[i].remove();
                }
            addEntry(newUl, liTemplate, '餐盒配送', '人均', feePerCustomer.toFixed(2));
            addEntry(newUl, liTemplate, '合计', '', cost.toFixed(2));
            addEntry(newUl, liTemplate, '<b>折后价</b>', 'x' + (rate).toFixed(2), (discountedCost).toFixed(2));
            x.appendChild(newUl);

            customers.push({
                name: getTitleEl(x).innerText,
                cost: discountedCost
            });
        });
    }

    function displaySubtotal () {
        var newItem = otherFee.cloneNode(true);
        var title = getTitleEl(newItem);
        if (!title) {
            throw err('找不到其它费用的标题');
        }
        title.innerText = '小计';
        var ul = newItem.querySelector('ul');
        var li = ul.querySelector('li');
        ul.innerHTML = '';
        customers.forEach(({name, cost}) => {
            addEntry(ul, li, name, '折后价', cost.toFixed(2));
        });
        rootEl.insertBefore(newItem, otherFee.nextSibling);
        /* 移除 [和Ta点一样] */
        var followEl = newItem.querySelector('header>div:last-child');
        if (followEl) {
            followEl.remove();
        }
    }

    function main () {
        if (window.elemePluginExecuted) {
            return;
        }
        getEls();
        computeTotal();
        parseCustomers();
        displaySubtotal();
        appendFooter();
        window.elemePluginExecuted = true;
    }
    main();
})();
```

## 费用问题

本插件已尽可能对价格进行准确计算，计算结果通常会比人工计算更加公平，但可能存在几个即使是人工计算也算不清的小问题：
 
 - 餐盒费用为所有人均摊，如果有人使用了比其它人更多的餐盒将导致计价发生误差。
 - 有些品类本身可享受在线支付立减之类的优惠，但如果订单发起人结算时使用了红包将导致总订单价更低但是这些优惠被取消。这些品类即使算上红包优惠价格也会比单点的立减优惠更贵，这种情况下的损失难以向其它人索赔。
 - 特价秒杀的商品和上面的情况类似，秒杀获得的折扣会被摊平到订单总折扣中，这种情况下的损失难以向其它人索赔。
 
<hr>

[编辑此页](https://github.com/jareguo/ele-me-web-extend/edit/master/README.md)<br>
[编辑网页](https://github.com/jareguo/ele-me-web-extend/edit/master/_layouts/default.html)<br>
[encodeURI online](http://pressbin.com/tools/urlencode_urldecode/)
