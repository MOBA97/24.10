/*** === CONFIG === ***/
const PORTAL_SHEET = 'Portal';     // key | salt | passhash | enabled
const USERS_SHEET  = 'GateUsers';  // name | role | enabled | doc_id | sheet | can_edit | salt | passhash

function doGet() {
  return HtmlService.createTemplateFromFile('Index')
    .evaluate()
    .setTitle('Вход')
    .setXFrameOptionsMode(HtmlService.XFrameOptionsMode.ALLOWALL);
}

/*** === UTILS === ***/
function sha256Hex_(s) {
  const raw = Utilities.computeDigest(Utilities.DigestAlgorithm.SHA_256, s);
  return raw.map(b => (b < 0 ? b + 256 : b).toString(16).padStart(2, '0')).join('');
}
function openSS_(){ return SpreadsheetApp.getActive(); }

function sheetUrlByIdAndName_(docId, sheetName) {
  const ss = SpreadsheetApp.openById(docId);
  if (!sheetName) return `https://docs.google.com/spreadsheets/d/${docId}/edit`;
  const sh = ss.getSheetByName(sheetName);
  if (!sh) throw new Error('Лист не найден: ' + sheetName);
  return `https://docs.google.com/spreadsheets/d/${docId}/edit#gid=${sh.getSheetId()}`;
}

/*** === LOADERS === ***/
function loadPortal_() {
  const sh = openSS_().getSheetByName(PORTAL_SHEET);
  if (!sh) throw new Error('Нет листа ' + PORTAL_SHEET);
  const v = sh.getDataRange().getValues();
  const head = v.shift().map(h => String(h).trim().toLowerCase());
  const ix = n => head.indexOf(n);
  const row = v.find(r => String(r[ix('key')]).trim().toLowerCase()==='portal');
  if (!row) throw new Error('В Portal нет строки key=portal');
  return {
    enabled: String(row[ix('enabled')]).toUpperCase()==='TRUE',
    salt: String(row[ix('salt')]||'').trim(),
    passhash: String(row[ix('passhash')]||'').trim().toLowerCase()
  };
}

function loadUsersFull_() {
  const sh = openSS_().getSheetByName(USERS_SHEET);
  if (!sh) throw new Error('Нет листа ' + USERS_SHEET);
  const v = sh.getDataRange().getValues();
  const head = v.shift().map(h => String(h).trim().toLowerCase());
  const ix = n => head.indexOf(n);
  ['name','role','enabled','doc_id','can_edit','salt','passhash'].forEach(n=>{
    if (ix(n)<0) throw new Error('В GateUsers нет колонки: '+n);
  });
  const L = [];
  v.forEach(r=>{
    const name = String(r[ix('name')]||'').trim();
    if (!name) return;
    const doc_id = String(r[ix('doc_id')]||'').trim();
    const sheet  = ix('sheet')>=0 ? String(r[ix('sheet')]||'').trim() : '';
    let url = '';
    if (doc_id) {
      url = sheet ? sheetUrlByIdAndName_(doc_id, sheet)
                  : `https://docs.google.com/document/d/${doc_id}/edit`;
    }
    L.push({
      name,
      role: String(r[ix('role')]||'').trim().toLowerCase(),
      enabled: String(r[ix('enabled')]).toUpperCase()==='TRUE',
      doc_id, sheet, url,
      can_edit: String(r[ix('can_edit')]).toUpperCase()==='TRUE',
      salt: String(r[ix('salt')]||'').trim(),
      passhash: String(r[ix('passhash')]||'').trim().toLowerCase()
    });
  });
  return L;
}

/*** === API: ПОРТАЛ === ***/
function api_gateLogin(portalPassword) {
  try{
    const P = loadPortal_();
    if (!P.enabled) return {ok:false, msg:'Портал отключён'};
    const need = Boolean(P.salt && P.passhash);
    if (need) {
      const calc = sha256Hex_(P.salt + String(portalPassword||''));
      if (calc !== P.passhash) return {ok:false, msg:'Неверный пароль'};
    }
    const users = loadUsersFull_()
      .filter(u=>u.enabled)
      .map(u=>({ name:u.name, role:u.role, protected:Boolean(u.salt && u.passhash) }));
    return {ok:true, users};
  }catch(e){ return {ok:false, msg:'Ошибка сервера: '+e.message}; }
}

/*** === API: РОЛЕВЫЕ ВХОДЫ === ***/
function api_userOpen(name, password){
  try{
    const u = loadUsersFull_().find(x => x.enabled && x.name === String(name||'').trim());
    if (!u) return {ok:false, msg:'Пользователь недоступен'};
    if (u.salt && u.passhash) {
      const calc = sha256Hex_(u.salt + String(password||'')); 
      if (calc !== u.passhash) return {ok:false, msg:'Неверный пароль'};
    }
    if (u.role === 'admin') {
      const all = loadUsersFull_().map(z=>({
        name:z.name, role:z.role, enabled:z.enabled, doc_id:z.doc_id, sheet:z.sheet, can_edit:z.can_edit
      }));
      return {ok:true, admin:true, users:all};
    }
    if (!u.url) return {ok:false, msg:'Для пользователя не задан документ'};
    return {ok:true, url:u.url};
  }catch(e){ return {ok:false, msg:'Ошибка сервера: '+e.message}; }
}

/*** === API: АДМИН === ***/
function api_adminUpsert(payload){
  try{
    const sh = openSS_().getSheetByName(USERS_SHEET);
    const v = sh.getDataRange().getValues();
    const head = v[0].map(h=>String(h).trim().toLowerCase());
    const col = n => head.indexOf(n)+1;
    let row = -1;
    for (let r=1;r<v.length;r++){
      if (String(v[r][col('name')-1]).trim() === String(payload.currentName||payload.name).trim()){
        row = r+1; break;
      }
    }
    if (row<0) row = sh.getLastRow()+1;
    sh.getRange(row,col('name')).setValue(payload.name);
    sh.getRange(row,col('role')).setValue(payload.role||'viewer');
    sh.getRange(row,col('enabled')).setValue(Boolean(payload.enabled));
    sh.getRange(row,col('doc_id')).setValue(payload.doc_id||'');
    if (col('sheet')>0) sh.getRange(row,col('sheet')).setValue(payload.sheet||'');
    sh.getRange(row,col('can_edit')).setValue(Boolean(payload.can_edit));
    return {ok:true};
  }catch(e){ return {ok:false, msg:e.message}; }
}
function api_adminResetPassword(name, newPassword){
  try{
    const sh = openSS_().getSheetByName(USERS_SHEET);
    const v = sh.getDataRange().getValues();
    const head = v[0].map(h=>String(h).trim().toLowerCase());
    const iN=head.indexOf('name'), iS=head.indexOf('salt'), iH=head.indexOf('passhash');
    for (let r=1;r<v.length;r++){
      if (String(v[r][iN]).trim()===String(name).trim()){
        const salt = Utilities.getUuid().replace(/-/g,'').slice(0,16);
        const passhash = sha256Hex_(salt + String(newPassword||''));
        sh.getRange(r+1,iS+1).setValue(salt);
        sh.getRange(r+1,iH+1).setValue(passhash);
        return {ok:true};
      }
    }
    return {ok:false, msg:'Не найден'};
  }catch(e){ return {ok:false, msg:e.message}; }
}

/*** === MENU: salt+hash === ***/
function onOpen(){
  SpreadsheetApp.getUi()
    .createMenu('Gate-Auth')
    .addItem('Сгенерировать salt+hash…', 'menuSaltHash_')
    .addToUi();
}
function menuSaltHash_(){
  const ui = SpreadsheetApp.getUi();
  const res = ui.prompt('Введите пароль', ui.ButtonSet.OK_CANCEL);
  if (res.getSelectedButton()!==ui.Button.OK) return;
  const pwd = res.getResponseText(); 
  if (!pwd){ ui.alert('Пароль пуст'); return; }
  const salt = Utilities.getUuid().replace(/-/g,'').slice(0,16);
  const passhash = sha256Hex_(salt + pwd);
  ui.alert('Готово', `salt: ${salt}\npasshash: ${passhash}\n\nВставьте в Portal или GateUsers.`, ui.ButtonSet.OK);
}
