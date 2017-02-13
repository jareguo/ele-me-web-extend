# ele-me-web-extend

面向 Chrome 的饿了么订单网页插件…… 帮你算出每个人要付多少，能统计餐盒、红包等费用。
在收藏夹中新增一个网页，将网址设为下面的完整代码。然后打开任一订单页面（需已下单），接着直接在收藏夹点击刚刚收藏的网址即可。

```js
javascript:(function () {
    var Yuan = '¥';
    var rootEl, customerEls, otherFee, totalEl;
    var feePerCustomer, rate;
    var customers = [];

    function err (text) {
        text = '插件错误: ' + text;
        alert(text);
        return new Error(text);
    }

    function getEls () {
        rootEl = document.body.querySelector('div.spell-items');
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
            // <div class="spell-total">
            // 5人，32份商品<span>合计 ¥83.05</span></div>
            throw err('找不到"合计"所在元素，订单可能已完成');
        }

    }
    function getCostEls (el) {
        return el.querySelectorAll('li>:last-child');
    }
    function addEntry (parent, template, title, unit, cost) {
        var newEl = template.cloneNode(true);
        var items = newEl.querySelectorAll('span');
        if (items.length < 3) {
            throw err('找不到单元格');
        }
        items[0].innerHTML = title;
        items[1].innerText = unit;
        items[1].style.width = '3rem';
        items[2].innerText = Yuan + cost;
        parent.appendChild(newEl);
    }
    function diplayCustomerCost (el, feePerCustomer, cost, rate, discountedCost) {
        var newUl = document.createElement('ul');
        newUl.classList = el.querySelector('ul').classList;
        var template = el.querySelector('li');
        addEntry(newUl, template, '餐盒配送', '人均', feePerCustomer);
        addEntry(newUl, template, '合计', '', cost);
        addEntry(newUl, template, '<b>折后价</b>', 'x' + (rate).toFixed(2), (discountedCost).toFixed(2));
        el.appendChild(newUl);
    }
    function getTitleEl (el) {
        return el.querySelector('header span:last-child');
    }
    function getCost (el) {
        var text = el.innerText.split(Yuan).splice(-1)[0];
        var cost = parseInt(text);
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
        // 1. 实付 / (实付 + 优惠)，算出折扣
        var total = getCost(totalEl);
        rate = total / (total + discount);
        // 2. 把其它支出费用均摊到每用户
        feePerCustomer = fee / customerEls.length;
    }

    function parseCustomers () {
        // 3. 每用户应付 * 折扣，算出每用户实付
        customerEls.forEach((x) => {
            var { fee, discount } = sumFee(x);
            var cost = (fee - discount + feePerCustomer);
            var discountedCost = cost * rate;
            console.log(discountedCost);

            diplayCustomerCost(x, feePerCustomer, cost, rate, discountedCost);

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
    }

    function main () {
        if (window.elemePluginExecuted) {
            return;
        }
        getEls();
        computeTotal();
        parseCustomers();
        displaySubtotal();
        window.elemePluginExecuted = true;
    }
    main();
})();
```
