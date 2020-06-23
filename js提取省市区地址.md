## js提取省市区地址

收到一个小需求，要实现用户粘贴一串复制出来的地址然后提取出正确的省市区，简单记录下，主要需要注意`盟`，`旗`这些单位

```js
const regList = [
  /.+?(省|自治区)/g, // 省
  /.+?(市|自治州|盟)/g, // 市
  /.+?区|.+?县|.+?旗/g, // 区
];

const body = [];
const getArea = (t, i) => {
  if (!regList[i]) {
    body[i] = t;
    return;
  }
  const reg = new RegExp(regList[i]);
  body[i] = t.match(reg) && t.match(reg)[0];
  const newText = t.replace(body[i], '');
  getArea(newText, i += 1);
};

getArea(text, 0);
console.log(body);
```