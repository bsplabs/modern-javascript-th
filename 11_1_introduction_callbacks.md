## บทนำ: Callbacks

ในตัวอย่างนี้ เราจะใช้ Browser methods เป็นหลัก เพื่อแสดงให้เห็นถึงการใช้งาน **Callbacks**, **Promises** และแนวคิดที่เป็นนามธรรมอื่นๆ โดยเราจะเน้นไปที่การโหลด Script และการจัดการ DOM แบบง่ายๆ

หากคุณยังไม่คุ้นเคยกับ Method เหล่านี้ และรู้สึกสับสนกับตัวอย่างที่ใช้ แนะนำให้ลองอ่านบทเนื้อหาในส่วนถัดไปของ Tutorial นี้ก่อน

อย่างไรก็ตาม เราจะพยายามอธิบายให้ชัดเจนที่สุด ซึ่งในส่วนของ Browser นั้นจะไม่มีอะไรซับซ้อนเกินไปครับ

Environment ที่รัน JavaScript (Host environments) ส่วนใหญ่จะมี Function ที่ช่วยให้เราสามารถจัดการ **Asynchronous actions** ได้ หรือพูดง่ายๆ คือ Action ที่เราสั่งให้เริ่มทำงาน "ตอนนี้" แต่จะไปเสร็จสิ้นใน "ภายหลัง"

ตัวอย่างที่เห็นได้ชัดคือ Function `setTimeout`

นอกจากนี้ยังมีตัวอย่างจริงๆ ของ Asynchronous actions อื่นๆ เช่น การโหลด Script และ Module (ซึ่งเราจะเจาะลึกกันในบทต่อๆ ไป)

ลองมาดู Function `loadScript(src)` ที่ทำหน้าที่โหลด Script จาก `src` ที่กำหนด:

```javascript
function loadScript(src) {
  // สร้าง tag <script> และ append เข้าไปใน page
  // ทำให้ script เริ่มโหลดและทำงานเมื่อโหลดเสร็จ
  let script = document.createElement('script');
  script.src = src;
  document.head.append(script);
}

```

Function นี้จะสร้าง Tag `<script src="...">` ขึ้นมาแบบ Dynamic และใส่ลงไปใน Document เมื่อ Browser ตรวจพบ มันจะเริ่มโหลด Script และ Execute ทันทีที่โหลดเสร็จ

เราสามารถเรียกใช้ Function นี้ได้แบบนี้:

```javascript
// โหลดและ execute script ตาม path ที่ระบุ
loadScript('/my/script.js');

```

ตัว Script จะถูก Execute แบบ **"Asynchronously"** เพราะมันเริ่มโหลดตอนนี้ แต่ไปรันจริงในภายหลังตอนที่ตัว Function หลักอาจจะทำงานจบไปแล้วด้วยซ้ำ

หากมี Code ต่อท้าย `loadScript(...)` มันจะไม่รอจนกว่า Script จะโหลดเสร็จครับ

```javascript
loadScript('/my/script.js');
// code ที่อยู่ใต้ loadScript
// จะไม่รอให้ script โหลดเสร็จก่อน แต่จะทำงานต่อทันที
// ...

```

สมมติว่าเราต้องการใช้งาน Script ตัวใหม่ทันทีที่มันโหลดเสร็จ (เช่น Script นั้นประกาศ Function ใหม่ๆ ไว้ และเราต้องการเรียกใช้)

แต่ถ้าเราเรียกใช้ทันทีหลังจากสั่ง `loadScript(...)` มันจะ Error ครับ:

```javascript
loadScript('/my/script.js'); // ใน script มี "function newFunction() {…}"

newFunction(); // Error: no such function!

```

แน่นอนว่า Browser มักจะยังโหลด Script ไม่เสร็จในเสี้ยววินาทีนั้น ปัญหาคือตอนนี้ `loadScript` ยังไม่มีวิธีที่บอกให้เรารู้ว่าโหลดเสร็จหรือยัง เราแค่อยากรู้ว่ามันเสร็จตอนไหน เพื่อที่จะได้เรียกใช้ Function หรือ Variable จาก Script นั้นได้

งั้นเราลองเพิ่ม **Callback function** เป็น Argument ตัวที่สองเพื่อให้มันทำงานเมื่อโหลดเสร็จกันครับ:

```javascript
function loadScript(src, callback) {
  let script = document.createElement('script');
  script.src = src;

  script.onload = () => callback(script);

  document.head.append(script);
}

```

Event `onload` (ซึ่งมีอธิบายในบท Resource loading: onload and onerror) จะทำหน้าที่ Execute function หลังจากที่ Script ถูกโหลดและประมวลผลเรียบร้อยแล้ว

คราวนี้ถ้าเราอยากเรียก Function ใหม่จาก Script เราก็แค่เขียนไว้ใน Callback:

```javascript
loadScript('/my/script.js', function() {
  // callback จะทำงานหลังจาก script โหลดเสร็จ
  newFunction(); // ตอนนี้ใช้งานได้แล้ว
  ...
});

```

นี่คือ Concept หลักครับ: Argument ตัวที่สองคือ Function (มักจะเป็น Anonymous function) ที่จะถูกสั่งให้รันเมื่อ Action นั้นๆ เสร็จสิ้น (Completed)

มาลองดูตัวอย่างที่รันได้จริงกับ Script ของจริงกัน:

```javascript
function loadScript(src, callback) {
  let script = document.createElement('script');
  script.src = src;
  script.onload = () => callback(script);
  document.head.append(script);
}

loadScript('https://cdnjs.cloudflare.com/ajax/libs/lodash.js/3.2.0/lodash.js', script => {
  alert(`Cool, the script ${script.src} is loaded`);
  alert( _ ); // _ คือ function ที่ประกาศไว้ใน script ที่โหลดมา
});

```

เราเรียกรูปแบบนี้ว่า **"Callback-based"** style ของการเขียนโปรแกรมแบบ Asynchronous คือ Function ใดก็ตามที่ทำงานแบบ Asynchronous ควรจะมี Argument สำหรับ Callback เพื่อให้เราใส่ Code ที่ต้องการให้รันหลังจบงานเข้าไปได้

ในที่นี้เรายกตัวอย่างด้วย `loadScript` แต่นี่คือแนวทางทั่วไปที่ใช้กันอย่างแพร่หลายครับ

---

## Callback in callback

แล้วถ้าเราอยากโหลด Script สองตัวต่อกันล่ะ? (ตัวแรกเสร็จก่อน แล้วค่อยต่อด้วยตัวที่สอง)

วิธีที่ทำได้เลยคือการใส่ `loadScript` ตัวที่สองไว้ใน Callback ของตัวแรก แบบนี้ครับ:

```javascript
loadScript('/my/script.js', function(script) {

  alert(`Cool, the ${script.src} is loaded, let's load one more`);

  loadScript('/my/script2.js', function(script) {
    alert(`Cool, the second script is loaded`);
  });

});

```

เมื่อ `loadScript` ตัวนอกทำงานเสร็จ Callback ก็จะไปสั่งตัวในให้เริ่มทำงานต่อ

แล้วถ้าอยากได้อีกสัก Script ล่ะ...?

```javascript
loadScript('/my/script.js', function(script) {

  loadScript('/my/script2.js', function(script) {

    loadScript('/my/script3.js', function(script) {
      // ...ทำต่อหลังจาก script ทั้งหมดโหลดเสร็จ
    });

  });

});

```

กลายเป็นว่าทุกๆ Action ใหม่จะต้องไปอยู่ใน Callback เรื่อยๆ ซึ่งถ้ามีแค่ 2-3 อันก็ยังพอไหว แต่ถ้ามีเยอะกว่านี้จะเริ่มลำบากแล้วครับ เดี๋ยวเราจะไปดูทางเลือกอื่นกันในภายหลัง

---

## Handling errors

ในตัวอย่างข้างบนเรายังไม่ได้พูดถึงเรื่อง Error เลย ถ้าเกิด Script โหลดไม่ผ่านล่ะ? Callback ของเราควรจะรับมือกับเหตุการณ์นั้นได้ด้วย

นี่คือ `loadScript` เวอร์ชั่นที่ปรับปรุงให้รองรับการเช็ค Error:

```javascript
function loadScript(src, callback) {
  let script = document.createElement('script');
  script.src = src;

  script.onload = () => callback(null, script);
  script.onerror = () => callback(new Error(`Script load error for ${src}`));

  document.head.append(script);
}

```

มันจะเรียก `callback(null, script)` เมื่อโหลดสำเร็จ และเรียก `callback(error)` เมื่อเกิดปัญหา

การนำไปใช้งาน:

```javascript
loadScript('/my/script.js', function(error, script) {
  if (error) {
    // จัดการ error ตรงนี้
  } else {
    // script โหลดสำเร็จ ทำงานต่อได้
  }
});

```

รูปแบบที่เราใช้กับ `loadScript` นี้เป็นรูปแบบที่เจอบ่อยมาก เรียกว่า **"Error-first callback"** style ครับ

โดยมีกฎ (Convention) คือ:

1. Argument ตัวแรกของ Callback จะถูกจองไว้สำหรับ **Error** (ถ้ามี) แล้วค่อยเรียก `callback(err)`
2. Argument ตัวที่สอง (และตัวถัดๆ ไป) จะใช้สำหรับผลลัพธ์ที่สำเร็จ แล้วค่อยเรียก `callback(null, result1, result2…)`

ดังนั้น Callback function เพียงอันเดียวจะทำหน้าที่ทั้งแจ้ง Error และส่งผลลัพธ์กลับมา

---

## Pyramid of Doom

มองแวบแรก วิธีนี้ก็ดูเป็นทางเลือกที่ดีสำหรับการเขียน Asynchronous code นะครับ และมันก็ใช้งานได้จริงๆ ถ้ามีการ Nested (ซ้อนกัน) แค่ 1 หรือ 2 ชั้นก็ดูไม่มีปัญหาอะไร

แต่ถ้ามี Asynchronous actions หลายตัวที่ต้องทำต่อกันไปเรื่อยๆ Code จะออกมาหน้าตาแบบนี้ครับ:

```javascript
loadScript('1.js', function(error, script) {

  if (error) {
    handleError(error);
  } else {
    // ...
    loadScript('2.js', function(error, script) {
      if (error) {
        handleError(error);
      } else {
        // ...
        loadScript('3.js', function(error, script) {
          if (error) {
            handleError(error);
          } else {
            // ...ทำต่อหลังจาก script ทุกตัวโหลดเสร็จ (*)
          }
        });

      }
    });
  }
});

```

จาก Code ด้านบน:

1. โหลด `1.js` ถ้าไม่มี error...
2. โหลด `2.js` ถ้าไม่มี error...
3. โหลด `3.js` ถ้าไม่มี error... ก็ทำอย่างอื่นต่อ (*)

พอมันมีการซ้อนกันลึกขึ้นเรื่อยๆ Code จะเริ่ม "เยื้องขวา" และจัดการได้ยากมาก โดยเฉพาะถ้ามี Logic จริงๆ แทนที่ไอ้เจ้า `...` เช่น มี Loop หรือเงื่อนไข `if-else` เพิ่มเข้ามาอีก

เราเรียกสิ่งนี้ว่า **"Callback hell"** หรือ **"Pyramid of doom"** ครับ

เมื่อการซ้อนกัน (Nested calls) ขยายไปทางขวาเรื่อยๆ ทุกครั้งที่มี Asynchronous action เพิ่มเข้ามา ในที่สุดมันก็จะหลุดจากการควบคุม

ดังนั้น วิธีการเขียน Code แบบนี้จึงไม่ค่อยดีนักในระยะยาว

เราอาจจะแก้ปัญหาเบื้องต้นได้ด้วยการแยกแต่ละ Action ออกมาเป็น Function ต่างหาก แบบนี้ครับ:

```javascript
loadScript('1.js', step1);

function step1(error, script) {
  if (error) {
    handleError(error);
  } else {
    // ...
    loadScript('2.js', step2);
  }
}

function step2(error, script) {
  if (error) {
    handleError(error);
  } else {
    // ...
    loadScript('3.js', step3);
  }
}

function step3(error, script) {
  if (error) {
    handleError(error);
  } else {
    // ...ทำต่อหลังจาก script ทุกตัวโหลดเสร็จ (*)
  }
}

```

เห็นไหมครับ? มันทำงานเหมือนเดิมเลย แต่ไม่มีการ Nested ลึกๆ แล้ว เพราะเราแยกแต่ละ Action ออกเป็น Function ระดับบน (Top-level)

แต่มันก็ยังมีข้อเสียอยู่ คือ Code มันดู "กระจัดกระจาย" เหมือน Spreadsheet ที่โดนฉีกออกเป็นชิ้นๆ อ่านยาก และคุณคงสังเกตเห็นว่าสายตาเราต้อง "กระโดด" ไปมาเพื่อไล่ดูการทำงานของ Code ซึ่งไม่สะดวกเลย โดยเฉพาะถ้าคนอ่านไม่คุ้นเคยกับ Code ชุดนี้มาก่อน เขาจะงงว่าต้องกระโดดไปอ่านตรงไหนต่อ

นอกจากนี้ Function ตระกูล `step*` พวกนี้ยังถูกสร้างมาเพื่อรันครั้งเดียวทิ้ง เพียงเพื่อเลี่ยง "Pyramid of doom" เท่านั้น ไม่ได้เอาไป Reuse ที่ไหนได้เลย ทำให้เกิด Namespace cluttering (ขยะในชื่อตัวแปร/ฟังก์ชัน) โดยใช่เหตุ

เราย่อมต้องการวิธีที่ดีกว่านี้

โชคดีที่ยังมีวิธีอื่นๆ ที่ช่วยหลีกเลี่ยง Pyramid เหล่านี้ได้ หนึ่งในวิธีที่ดีที่สุดคือการใช้ **"Promises"** ซึ่งเราจะคุยกันในบทถัดไปครับ