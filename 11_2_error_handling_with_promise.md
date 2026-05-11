# Error handling with promises

Promise chains มีประสิทธิภาพดีเมื่อจัดการ error ได้ดีเยี่ยม เมื่อ promise ถูก reject control จะกระโดดไปยัง rejection handler ที่ใกล้ที่สุด ซึ่งสะดวกมากในการใช้งานจริง

ตัวอย่างเช่น ในโค้ดด้านล่าง URL ที่ fetch มีปัญหา (เซิร์ฟเวอร์ไม่มีอยู่จริง) และ .catch จะจัดการ error:

```javascript
fetch('https://no-such-server.blabla') // rejects
  .then(response => response.json())
  .catch(err => alert(err)) // TypeError: failed to fetch (ข้อความอาจต่างกันไป)
```

อย่างที่เห็น .catch ไม่จำเป็นต้องมาทันทีหลัง promise ที่ reject มันอาจปรากฏหลัง .then หนึ่งตัวหรือหลายตัวก็ได้

หรือบางที เซิร์ฟเวอร์ทำงานปกติทั้งหมด แต่ response ไม่ใช่ valid JSON วิธีที่ง่ายที่สุดในการ catch error ทั้งหมดคือการเพิ่ม .catch ที่ท้ายสุดของ chain:

```javascript
fetch('/article/promise-chaining/user.json')
  .then(response => response.json())
  .then(user => fetch(`https://api.github.com/users/${user.name}`))
  .then(response => response.json())
  .then(githubUser => new Promise((resolve, reject) => {
    let img = document.createElement('img');
    img.src = githubUser.avatar_url;
    img.className = "promise-avatar-example";
    document.body.append(img);

    setTimeout(() => {
      img.remove();
      resolve(githubUser);
    }, 3000);
  }))
  .catch(error => alert(error.message));
```

ปกติแล้ว .catch แบบนี้จะไม่ถูก trigger เลย แต่ถ้าหากว่า promise ใดๆ ด้านบนถูก reject (ปัญหาเครือข่าย หรือ JSON ไม่ valid หรืออื่นๆ) มันจะ catch ได้

## Implicit try…catch

โค้ดใน promise executor และ promise handlers มี "invisible try..catch" ห้อมรอบไว้ ถ้ามี exception เกิดขึ้น มันจะถูก caught และถือว่าเป็น rejection

ตัวอย่างเช่น โค้ดนี้:

```javascript
new Promise((resolve, reject) => {
  throw new Error("Whoops!");
}).catch(alert); // Error: Whoops!
```

ทำงานเหมือนกับนี้:

```javascript
new Promise((resolve, reject) => {
  reject(new Error("Whoops!"));
}).catch(alert); // Error: Whoops!
```

"Invisible try..catch" รอบ executor จะ auto catch error และเปลี่ยนเป็น rejected promise

สิ่งนี้ไม่เกิดขึ้นในฟังก์ชัน executor เท่านั้น แต่ในหลาย handlers ด้วย ถ้าเราทำ throw ใน .then handler นั่นหมายถึง promise ที่ถูก reject ดังนั้น control จะกระโดดไปยัง error handler ที่ใกล้ที่สุด

นี่คือตัวอย่าง:

```javascript
new Promise((resolve, reject) => {
  resolve("ok");
}).then((result) => {
  throw new Error("Whoops!"); // rejects the promise
}).catch(alert); // Error: Whoops!
```

สิ่งนี้เกิดขึ้นสำหรับ error ทั้งหมด ไม่เพียงแค่ที่เกิดจาก throw statement เท่านั้น ตัวอย่างเช่น programming error:

```javascript
new Promise((resolve, reject) => {
  resolve("ok");
}).then((result) => {
  blabla(); // no such function
}).catch(alert); // ReferenceError: blabla is not defined
```

.catch ที่อยู่ท้ายสุด ไม่เพียงแต่ catch explicit rejection เท่านั้น แต่ยัง catch accidental error ใน handlers ด้านบนได้ด้วย

## Rethrowing

อย่างที่เราสังเกตเห็น .catch ที่ท้าย chain มีความคล้ายกับ try..catch ธรรมดา เราอาจมี .then handlers ได้หลายตัว แล้วใช้ .catch เพียงตัวเดียวที่ท้าย เพื่อจัดการ error ใน handlers ทั้งหมด

ใน regular try..catch เราสามารถ analyze error และบางที rethrow ได้ถ้า handle ไม่ได้ สิ่งเดียวกันก็สามารถทำได้กับ promises

ถ้าเราทำ throw ใน .catch ดังนั้น control จะไปยัง error handler ที่ใกล้ที่สุดตัวถัดไป และถ้าเราจัดการ error และจบลงอย่างปกติ มันจะไปยัง successful .then handler ที่ใกล้ที่สุดตัวถัดไป

ในตัวอย่างด้านล่าง .catch สามารถจัดการ error ได้สำเร็จ:

```javascript
// the execution: catch -> then
new Promise((resolve, reject) => {

  throw new Error("Whoops!");

}).catch(function(error) {

  alert("The error is handled, continue normally");

}).then(() => alert("Next successful handler runs"));
```

ที่นี่ .catch block จบลงอย่างปกติ ดังนั้น next successful .then handler จึงถูกเรียก

ในตัวอย่างด้านล่าง เราเห็นอีกสถานการณ์หนึ่งกับ .catch handler (*) catch error แต่ก็ handle ไม่ได้ (เช่น มันรู้แค่ว่า handle URIError ได้) ดังนั้นมันจึง throw ใหม่:

```javascript
// the execution: catch -> catch
new Promise((resolve, reject) => {

  throw new Error("Whoops!");

}).catch(function(error) { // (*)

  if (error instanceof URIError) {
    // handle it
  } else {
    alert("Can't handle such error");

    throw error; // throwing this or another error jumps to the next catch
  }

}).then(function() {
  /* doesn't run here */
}).catch(error => { // (**)

  alert(`The unknown error has occurred: ${error}`);
  // don't return anything => execution goes the normal way

});
```

Execution กระโดดจาก first .catch (*) ไปยัง next one (**) ลงมาตามด้านล่างของ chain

## Unhandled rejections

เกิดอะไรขึ้นเมื่อ error ไม่ได้ถูก handle ตัวอย่างเช่น เราลืม append .catch ที่ท้าย chain แบบนี้:

```javascript
new Promise(function() {
  noSuchFunction(); // Error here (no such function)
})
  .then(() => {
    // successful promise handlers, one or more
  }); // without .catch at the end!
```

ในกรณีที่มี error promise จะกลายเป็น rejected และ execution ควรกระโดดไปยัง closest rejection handler แต่ไม่มี ดังนั้น error จึงติดค้าง (stuck) ไม่มีโค้ดที่จะ handle มัน

ในการใช้งานจริง เหมือนกับ unhandled error ธรรมดาใน code มันหมายความว่ามีบางอย่างผิดพลาดไปแล้ว

เกิดอะไรขึ้นเมื่อ regular error เกิดขึ้นและไม่ถูก caught โดย try..catch script จะตายพร้อมกับข้อความใน console สิ่งเดียวกันเกิดขึ้นกับ unhandled promise rejection

JavaScript engine track rejections เหล่านี้และสร้าง global error ในกรณีนั้น คุณสามารถเห็นได้ใน console ถ้าคุณรัน example ด้านบน

ในเบราว์เซอร์ เราสามารถ catch error เหล่านี้ได้โดยใช้ event `unhandledrejection`:

```javascript
window.addEventListener('unhandledrejection', function(event) {
  // the event object has two special properties:
  alert(event.promise); // [object Promise] - the promise that generated the error
  alert(event.reason); // Error: Whoops! - the unhandled error object
});

new Promise(function() {
  throw new Error("Whoops!");
}); // no catch to handle the error
```

Event นี้เป็นส่วนของ HTML standard

ถ้า error เกิดขึ้นและไม่มี .catch unhandledrejection handler จะถูก trigger และจะได้รับ event object ที่มีข้อมูลเกี่ยวกับ error เพื่อให้เราทำบางอย่างได้

ปกติแล้ว error เหล่านี้ไม่สามารถกู้คืนได้ ดังนั้นทางออกที่ดีที่สุดของเราคือบอกให้ user ทราบเกี่ยวกับปัญหา และอาจจะรายงานเหตุการณ์ไปยังเซิร์ฟเวอร์

ในสภาพแวดล้อมที่ไม่ใช่เบราว์เซอร์เช่น Node.js มีวิธีอื่นๆ ในการ track unhandled error

## Summary

- .catch จัดการ error ใน promises ทุกชนิด: ไม่ว่าจะเป็น reject() call หรือ error ที่ throw ใน handler
- .then ก็ catch error ในลักษณะเดียวกัน ถ้าได้รับ second argument (ซึ่งเป็น error handler)
- เราควรวาง .catch ตรงจากที่เราต้องการจัดการ error และรู้วิธีจัดการมัน Handler ควรวิเคราะห์ error (custom error classes ช่วยได้) และ rethrow unknown ones (บางที อาจเป็น programming mistake)
- ไม่เป็นไรถ้าไม่ใช้ .catch เลย ถ้าไม่มีทางที่จะกู้คืน error ได้
- ในกรณีใดก็ตาม เราควรมี unhandledrejection event handler (สำหรับเบราว์เซอร์ และ analogs สำหรับสภาพแวดล้อมอื่นๆ) เพื่อ track unhandled error และบอกให้ user ทราบ (และอาจจะเซิร์ฟเวอร์ของเรา) เกี่ยวกับมัน เพื่อไม่ให้ app ของเรา "แค่ตาย" ไปเลย

## Tasks

### Error in setTimeout

คิดดูสิ .catch จะ trigger ไหม อธิบายคำตอบของคุณ

```javascript
new Promise(function(resolve, reject) {
  setTimeout(() => {
    throw new Error("Whoops!");
  }, 1000);
}).catch(alert);
```

#### Solution

คำตอบคือ: ไม่ จะไม่ trigger:

```javascript
new Promise(function(resolve, reject) {
  setTimeout(() => {
    throw new Error("Whoops!");
  }, 1000);
}).catch(alert);
```

อย่างที่บอกไว้ในบท มี "implicit try..catch" รอบโค้ดของฟังก์ชัน ดังนั้น synchronous error ทั้งหมดจึงถูก handle

แต่ที่นี่ error เกิดขึ้นไม่ใช่ขณะที่ executor กำลังทำงาน แต่เกิดขึ้นทีหลัง ดังนั้น promise จึงไม่สามารถ handle มันได้