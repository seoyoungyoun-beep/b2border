# ABLE B2B 주문폼 (자체 페이지) — HTML + Google Apps Script (싱글오리진 드롭다운, 슬랙 연동)

- 품목 리스트: 제공해주신 스프레드시트 반영
- 싱글 오리진: **200g / 1kg 드롭다운 선택 + 제품명 직접 입력 가능**
- 안내문: 상단에 안내사항 2줄 반영
- 알림메일은 제외, 대신 슬랙 Webhook 호출 코드 추가
- 질문 흐름은 “매장 이름 → 제품명·수량 → 연락처” 순으로 간소화
- 마지막에 주문 요약을 **카카오톡 알림톡 대신, 우선 슬랙 Webhook 메시지**로 전송 (추후 알림톡은 카카오 API 연동 필요)

---

## 1) index.html

```html
<!doctype html>
<html lang="ko">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1" />
  <title>ABLE COFFEE GROUP — B2B 주문서</title>
  <style>
    body {font-family: system-ui, -apple-system, Segoe UI, Roboto, Noto Sans KR, Helvetica, Arial, sans-serif; margin: 0; background:#fafafa; color:#111;}
    .container {max-width: 700px; margin: 24px auto; padding: 24px; background:#fff; border-radius: 16px; box-shadow: 0 8px 24px rgba(0,0,0,0.06);} 
    h1 {margin: 0 0 8px; font-size: 24px}
    .sub {color:#444; margin-bottom: 24px; white-space: pre-line;}
    label {font-size: 14px; color:#333; display:block; margin:12px 0 6px}
    input, select {width: 100%; padding: 10px 12px; border:1px solid #ddd; border-radius: 10px; font-size: 14px}
    input[type="number"] {text-align: right}
    table {width: 100%; border-collapse: collapse; margin-top:12px}
    th, td {padding: 10px; border-bottom: 1px solid #eee; text-align: left; font-size: 14px}
    th {background:#fafafa; font-weight: 600}
    .btn {appearance:none; border:none; border-radius: 10px; padding: 10px 14px; cursor:pointer; font-weight:600; margin-top:16px}
    .btn-primary {background:#111; color:#fff; float:right}
    .btn-danger {background:#ffe3e3; color:#c92a2a}
    .success {background:#e6fcf5; color:#087f5b; padding:12px 14px; border-radius:10px; margin-top:12px}
    .error {background:#fff5f5; color:#c92a2a; padding:12px 14px; border-radius:10px; margin-top:12px}
  </style>
</head>
<body>
  <div class="container">
    <h1>ABLE COFFEE GROUP — B2B 주문서</h1>
    <div class="sub">평일 오전 10시까지 주문시 당일 출고됩니다. 나머지 주문 건들은 익일 출고됩니다.\n긴급 출고 요청 시, 카톡문의 혹은 전화 주세요. (010 8767 4709)</div>

    <label>매장 이름</label>
    <input id="company" placeholder="예) 강남점 / 홍대점" />

    <label>주문 품목</label>
    <table id="itemsTable">
      <thead>
        <tr><th>제품명</th><th>규격</th><th>수량</th><th></th></tr>
      </thead>
      <tbody id="tbody"></tbody>
    </table>
    <button id="addRow" class="btn">+ 품목 추가</button>

    <label>연락처</label>
    <input id="phone" placeholder="010-1234-5678" />

    <button id="submitBtn" class="btn btn-primary">주문 제출</button>
    <div id="msg"></div>
  </div>

<script>
const ITEMS = [
  "[원두] 블랙수트_1kg",
  "[원두] 벨벳화이트_1kg",
  "[원두] 세븐티_1kg",
  "[원두] 몰트_1kg",
  "[원두] 매트_1kg",
  "[원두] 디카페인_ 미디엄로스트_1kg",
  "[원두] 디카페인_ 다크 로스트_1kg",
  "[콜드브루] 블랙수트(고농도)_1L",
  "[콜드브루] 블랙수트(중농도)_500ml",
  "[콜드브루] 디카페인(고농도)_1L",
  "[콜드브루] 벨벳화이트(중농도)_500ml",
  "[콜드브루] 디카페인(중농도)_500ml",
  "[콜드브루] 블랙수트 RTD_235ml",
  "[기타] 싱글 오리진_(제품명 변경)"
];

const SIZES = ["1kg", "1L", "500ml", "235ml"];
const SINGLE_SIZES = ["200g","1kg"];
const WEB_APP_URL = "YOUR_APPS_SCRIPT_WEB_APP_URL";

const tbody = document.getElementById('tbody');
function makeSelect(options){const s=document.createElement('select');options.forEach(o=>{let op=document.createElement('option');op.value=o;op.textContent=o;s.appendChild(op)});return s;}

function addRow(){
  const tr=document.createElement('tr');
  const tdItem=document.createElement('td');
  const selItem=makeSelect(ITEMS);
  const customName=document.createElement('input');
  customName.placeholder='싱글 오리진 제품명'; customName.style.display='none';
  tdItem.appendChild(selItem); tdItem.appendChild(customName);

  const tdSize=document.createElement('td');
  const selSize=makeSelect(SIZES);
  selSize.disabled=true;
  tdSize.appendChild(selSize);

  const tdQty=document.createElement('td');
  const qty=document.createElement('input'); qty.type='number'; qty.min='1'; qty.placeholder='0';
  tdQty.appendChild(qty);

  const tdDel=document.createElement('td');
  const del=document.createElement('button'); del.textContent='삭제'; del.className='btn btn-danger'; del.onclick=()=>tr.remove();
  tdDel.appendChild(del);

  tr.appendChild(tdItem); tr.appendChild(tdSize); tr.appendChild(tdQty); tr.appendChild(tdDel);
  tbody.appendChild(tr);

  function inferSize(label){const m=label.match(/_(\d+(?:kg|ml|L))$/)||label.match(/(\d+(?:kg|ml|L))$/);return m?m[1]:'';}

  function handleItemChange(){
    const v=selItem.value;
    if(v.includes('싱글 오리진')){
      customName.style.display='block';
      tdSize.innerHTML='';
      const singleSel=makeSelect(SINGLE_SIZES);
      tdSize.appendChild(singleSel);
    } else {
      customName.style.display='none';
      tdSize.innerHTML='';
      const sizeSel=makeSelect(SIZES);
      const s=inferSize(v); if(s) sizeSel.value=s; sizeSel.disabled=true;
      tdSize.appendChild(sizeSel);
    }
  }
  selItem.addEventListener('change',handleItemChange);
  handleItemChange();
}

addRow();
document.getElementById('addRow').onclick=addRow;

async function submitOrder(){
  const company=document.getElementById('company').value.trim();
  const phone=document.getElementById('phone').value.trim();
  if(!company||!phone){showError('매장명과 연락처를 입력해주세요.');return;}
  const rows=[...tbody.querySelectorAll('tr')];
  const items=[];
  for(const r of rows){
    const baseLabel=r.children[0].querySelector('select').value;
    const customName=r.children[0].querySelector('input');
    const size=r.children[1].querySelector('select').value;
    const qty=parseInt(r.children[2].querySelector('input').value||'0',10);
    if(qty>0){
      const name=baseLabel.includes('싱글 오리진')?(customName.value.trim()||'싱글 오리진'):baseLabel;
      items.push({item:name,size,qty});
    }
  }
  if(items.length===0){showError('최소 1개 이상 입력해주세요.');return;}
  const payload={company,phone,items};
  try{
    const res=await fetch(WEB_APP_URL,{method:'POST',headers:{'Content-Type':'application/json'},body:JSON.stringify(payload)});
    const data=await res.json();
    if(data.ok){showSuccess(`주문이 접수되었습니다. 주문번호: ${data.orderId}`);} else {showError('오류: '+data.message);}
  }catch(e){showError('네트워크 오류');}
}

function showSuccess(t){document.getElementById('msg').innerHTML=`<div class='success'>${t}</div>`;}
function showError(t){document.getElementById('msg').innerHTML=`<div class='error'>${t}</div>`;}

document.getElementById('submitBtn').onclick=submitOrder;
</script>
</body>
</html>
```

---

## 2) Google Apps Script Code.gs

```javascript
const SPREADSHEET_ID = '1hxxpers2CoLeFZdCn9aECnn9SfXEB08a67wQJHzPUoM';
const ORDERS_SHEET = 'Orders';
const ITEMS_SHEET = 'OrderItems';
const SLACK_WEBHOOK_URL = 'YOUR_SLACK_WEBHOOK_URL';

function doPost(e){
  try{
    const data=JSON.parse(e.postData.contents);
    const {company,phone,items}=data;
    if(!company||!phone||!items.length) return jsonResponse({ok:false,message:'필수값 누락'},400);

    const ss=SpreadsheetApp.openById(SPREADSHEET_ID);
    const orders=ss.getSheetByName(ORDERS_SHEET)||ss.insertSheet(ORDERS_SHEET);
    const details=ss.getSheetByName(ITEMS_SHEET)||ss.insertSheet(ITEMS_SHEET);
    if(orders.getLastRow()===0) orders.appendRow(['timestamp','orderId','company','phone','totalQty']);
    if(details.getLastRow()===0) details.appendRow(['timestamp','orderId','item','size','qty']);

    const orderId=generateOrderId(orders);
    const ts=new Date();
    const totalQty=items.reduce((a,b)=>a+(b.qty||0),0);
    orders.appendRow([ts,orderId,company,phone,totalQty]);
    const rows=items.map(it=>[ts,orderId,it.item,it.size,it.qty]);
    details.getRange(details.getLastRow()+1,1,rows.length,rows[0].length).setValues(rows);

    // Slack 알림
    const lines=items.map(i=>`- ${i.item} / ${i.size} × ${i.qty}`).join("\n");
    const payload={text:`[B2B 주문 접수] ${company}\n연락처: ${phone}\n총 수량: ${totalQty}\n\n${lines}\n\n주문번호: ${orderId}`};
    UrlFetchApp.fetch(SLACK_WEBHOOK_URL,{method:'post',contentType:'application/json',payload:JSON.stringify(payload)});

    return jsonResponse({ok:true,orderId});
  }catch(err){
    return jsonResponse({ok:false,message:err.message},500);
  }
}

function generateOrderId(sheet){
  const today=Utilities.formatDate(new Date(),Session.getScriptTimeZone(),'yyyyMMdd');
  let seq=1;
  if(sheet.getLastRow()>1){
    const values=sheet.getRange(2,2,sheet.getLastRow()-1,1).getValues().flat();
    const todayIds=values.filter(v=>(v||'').toString().startsWith(today));
    if(todayIds.length){
      const last=todayIds[todayIds.length-1];
      const m=String(last).match(/-(\d{3})$/);
      if(m) seq=parseInt(m[1],10)+1;
    }
  }
  return `${today}-${String(seq).padStart(3,'0')}`;
}

function jsonResponse(obj,status){
  return ContentService.createTextOutput(JSON.stringify(obj)).setMimeType(ContentService.MimeType.JSON).setResponseCode(status||200);
}
```

---

👉 이 상태에서 **슬랙 Webhook URL**만 교체하면, 주문 시 자동으로 시트에 기록되고 슬랙 알림이 옵니다.  
카카오 알림톡은 별도 API 연동이 필요해서, 차후 원하면 카카오 BizMessage API 붙이는 방법도 안내드릴 수 있어요.

