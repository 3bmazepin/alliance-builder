// ==UserScript==
// @name         同盟拠点 建設・削除プランナー v0.3.13
// @namespace    alliance-builder
// @version      0.3.13
// @description  施設と配置を選ぶ建設予定UI。建設操作は行いません。
// @match        https://*.3gokushi.jp/*
// @grant        GM_info
// ==/UserScript==

/* Shared parser and runner. Bundled into the installable userscript. */
(function (scope) {
  'use strict';
  // Prototype-era pages can add Array.prototype.toJSON returning a JSON string.
  // Serialize our plain data without invoking page-defined toJSON methods.
  function serialize(value){
    if(value===null)return 'null';
    if(Array.isArray(value)){const items=[];for(let i=0;i<value.length;i++)items.push(serialize(value[i])??'null');return '['+items.join(',')+']';}
    if(typeof value==='object')return '{'+Object.keys(value).filter(k=>value[k]!==undefined).map(k=>JSON.stringify(k)+':'+serialize(value[k])).join(',')+'}';
    return JSON.stringify(value);
  }
  const normalize = s => String(s || '').replace(/\s+/g, '').replace(/[･・]/g, '・');
  const coordOf = u => /^[0-6]$/.test(u.searchParams.get('x')) && /^[0-6]$/.test(u.searchParams.get('y')) ? `${u.searchParams.get('x')},${u.searchParams.get('y')}` : null;
  function linkURL(element,base){
    // Support HTML-escaped ampersands from older map templates, but never execute JS.
    const href=(element.getAttribute('href')||'').replace(/&amp;/g,'&');
    if(!href||/^(?:javascript:|#)/i.test(href))return null;
    return safeURL(href,base);
  }
  const denied = text => /(?:役職|権限)[^。\n]{0,60}(?:ありません|ないため|必要|できません|出来ません)|(?:操作|管理|建設)[^。\n]{0,30}権限[^。\n]{0,30}(?:ありません|必要)/.test(text);
  function jobFinish(text,now=Date.now()){
    const remaining=String(text).match(/あと\s*(\d+):(\d{2}):(\d{2})/);
    const clock=String(text).match(/(\d{1,2}):(\d{2}):(\d{2})\s*に完了/);
    const estimate=remaining?now+(Number(remaining[1])*3600+Number(remaining[2])*60+Number(remaining[3]))*1000:null;
    if(clock&&Number(clock[1])<24&&Number(clock[2])<60&&Number(clock[3])<60){
      const reference=estimate??now,day=Math.floor((reference+9*3600000)/86400000)*86400000-9*3600000;
      const target=day+(Number(clock[1])*3600+Number(clock[2])*60+Number(clock[3]))*1000;
      if(estimate!==null)return [target-86400000,target,target+86400000].sort((a,b)=>Math.abs(a-estimate)-Math.abs(b-estimate))[0];
      return target<now-300000?target+86400000:target;
    }
    return estimate;
  }
  function facilityAvailability(facility,state){
    if(!state)return {status:'unknown',reasons:['最新状態を取得していません']};
    const [name,required,limit,,prerequisite]=facility,reasons=[];
    const members=state.text.match(/同盟人数\s*([\d,]+)\s*\//);
    if(members&&Number(members[1].replace(/,/g,''))<required)reasons.push(`必要人数不足（${members[1]}／${required}人）`);
    const cells=[...state.cells.values()];
    if(cells.filter(c=>c.name===name).length>=limit)reasons.push(`建設上限（${limit}件）`);
    if(prerequisite&&!cells.some(c=>c.name===prerequisite&&!c.status&&!state.jobs.some(j=>j.coord===c.coord||(!j.coord&&j.name===prerequisite))))reasons.push(`前提施設未完成：${prerequisite}`);
    if(!cells.some(c=>c.empty))reasons.push('建築できる空きマスなし');
    return {status:reasons.length?'blocked':members?'candidate':'unknown',reasons:reasons.length?reasons:members?['人数・前提施設・個数の条件を満たしています。実行時にゲームの建設リンクを再確認します。']:['同盟人数を取得できていません']};
  }
  function safeURL(href, base) {
    const u = new URL(href, base);
    if (u.origin !== new URL(base).origin || !/^\/alliance\//.test(u.pathname) || !/^https:$/.test(u.protocol)) throw Error('同盟拠点以外の送信先は利用しません。');
    return u;
  }
  function parseVillage(doc, base, names) {
    // Callers historically supplied just the origin. Relative facility/... links
    // are relative to /alliance/village.php, never to the site's root directory.
    base=new URL('/alliance/village.php',base).href;
    const text = doc.body.textContent || '';
    if (denied(text)) throw Error('同盟拠点の操作権限がありません。役職を確認してください。');
    if (!/同盟拠点|建設状況/.test(text)) throw Error('同盟拠点画面を取得できません。ログイン状態を確認してください。');
    const cells = new Map(), jobs = [], trace={links:0,candidates:0,accepted:0,failures:[]};
    const findName = value => [...names].sort((a,b)=>b.length-a.length).find(n=>normalize(value).includes(normalize(n)));
    for (const a of doc.querySelectorAll('a[href],area[href]')) {
      trace.links++;const raw=a.getAttribute('href')||'';const candidate=/(?:^|\/)facility\//.test(raw);if(candidate)trace.candidates++;
      let u;try {u=linkURL(a,base);}catch(error){if(candidate&&trace.failures.length<8)trace.failures.push(error.message);continue;}if(!u)continue;
      const coord=coordOf(u);if(!coord || !/^\/alliance\/facility\//.test(u.pathname)){if(candidate&&trace.failures.length<8)trace.failures.push('URLの座標または施設パスを認識できません');continue;}
      trace.accepted++;
      const desc=[a.textContent,a.title,a.getAttribute('alt'),a.getAttribute('aria-label'),...Array.from(a.querySelectorAll('img')).flatMap(i=>[i.alt,i.title])].filter(Boolean).join(' ');
      const name=findName(desc), status=/削除中/.test(desc)?'delete':/建設準備中/.test(desc)?'prepare':/建設中/.test(desc)?'build':null;
      const empty=u.pathname==='/alliance/facility/select_facility.php'&&!name&&!status;
      const previous=cells.get(coord);
      if(!previous||!empty)cells.set(coord,{coord,name:name||previous?.name||null,empty,url:previous&&new URL(previous.url).pathname.endsWith('/facility.php')?previous.url:u.href,status:status||previous?.status||null});
    }
    // Only accept small, local job rows. A broad page container cannot identify a job.
    for(const node of doc.querySelectorAll('li,tr,div,p,span')) {
      const row=node.textContent.trim();
      if(row.length>260 || !/建設準備中|建設中|削除中/.test(row))continue;
      if((row.match(/建設準備中|建設中|削除中/g)||[]).length!==1)continue;
      if(Array.from(node.children).some(child=>/建設準備中|建設中|削除中/.test(child.textContent)&&findName(child.textContent)))continue;
      const kind=/削除中/.test(row)?'delete':/建設準備中/.test(row)?'prepare':'build';
      const name=findName(row);if(!name)continue;
      const coords=new Set();for(const a of node.querySelectorAll('[href]')){try{const c=coordOf(safeURL(a.getAttribute('href'),base));if(c)coords.add(c);}catch{}}
      const coord=coords.size===1?[...coords][0]:null;
      jobs.push({kind,name,coord,text:row});
    }
    for(const cell of cells.values())if(cell.status&&!jobs.some(j=>j.coord===cell.coord&&j.kind===cell.status))jobs.push({kind:cell.status,name:cell.name,coord:cell.coord,text:cell.status});
    // Legacy game pages often use plain text and <br> directly inside a large div.
    // Preserve visual line boundaries instead of requiring one DOM element per job.
    function lines(node){
      if(node.nodeType===3)return node.textContent;
      if(node.nodeType!==1)return '';
      if(/^(SCRIPT|STYLE|NOSCRIPT)$/.test(node.tagName))return '';
      if(node.tagName==='BR')return '\n';
      const content=Array.from(node.childNodes).map(lines).join('');
      return /^(DIV|P|LI|TR|TABLE|SECTION|H[1-6])$/.test(node.tagName)?'\n'+content+'\n':content;
    }
    const visualLines=lines(doc.body).split(/\n/).map(s=>s.trim()).filter(Boolean);
    for(let i=0;i<visualLines.length;i++){
      const line=visualLines[i];if(!/建設準備中|建設中|削除中/.test(line))continue;
      const kind=/削除中/.test(line)?'delete':/建設準備中/.test(line)?'prepare':'build';
      let row=line,name=findName(row);
      if(!name && /^(建設準備中|建設中|削除中)\s*[:：]?$/.test(line)){
        const next=visualLines[i+1]||'';if(!/建設準備中|建設中|削除中/.test(next)){row+=' '+next;name=findName(row);}
      }
      if(!name)continue;
      const existing=jobs.find(j=>j.kind===kind&&j.name===name);
      if(!existing)jobs.push({kind,name,coord:null,text:row});
      else if(jobFinish(row)!==null&&jobFinish(existing.text)===null)existing.text=row;
    }
    for(const job of jobs){
      job.finishAt=jobFinish(job.text);
      if(!job.coord){
        const candidates=[...cells.values()].filter(c=>c.name===job.name);
        if(candidates.length===1)job.coord=candidates[0].coord;
      }
      if(job.coord){const cell=cells.get(job.coord);if(cell){cell.status=job.kind;cell.empty=false;}}
    }
    const completeIdleBoard=cells.size===43&&/建設状況/.test(text)&&/建設件数/.test(text)&&!/建設準備中|建設中|削除中/.test(text);
    const jobsKnown=(completeIdleBoard || jobs.length>0 || /実行中の作業/.test(text) || /(?:建設|作業)(?:中の)?(?:はありません|していません)/.test(text)) && (!/建設準備中/.test(text)||jobs.some(j=>j.kind==='prepare')) && (!/削除中/.test(text)||jobs.some(j=>j.kind==='delete')) && (!/建設中/.test(text)||jobs.some(j=>j.kind==='build'));
    if(!cells.size){const error=Error('区画リンクを読み取れません。候補'+trace.candidates+'件／認識'+trace.accepted+'件。'+(trace.failures[0]||'区画リンクの形式を確認します。'));error.diagnostics=trace;throw error;}
    const fingerprint=serialize({cells:[...cells.values()].map(({coord,name,empty,status})=>({coord,name,empty,status})).sort((a,b)=>a.coord.localeCompare(b.coord)),jobs:jobs.map(({kind,name,coord})=>({kind,name,coord})).sort((a,b)=>serialize(a).localeCompare(serialize(b)))});
    return {cells,jobs,jobsKnown,fingerprint,text};
  }
  function reconcile(queue, state) {
    const claimed=new Set();
    return queue.map(item=>{
      const q={...item};if(q.phase==='done')return q;
      const isDelete=q.action==='delete', coord=q.coord, cell=state.cells.get(coord);
      const match=state.jobs.findIndex((job,index)=>!claimed.has(index)&&job.name===q.name&&(isDelete?job.kind==='delete':job.kind!=='delete')&&(coord?job.coord===coord:q.mode==='auto'));
      if(match>=0){claimed.add(match);q.phase='observed';q.reason='予定に一致する作業を確認（他の役職の操作も優先）';if(state.jobs[match].coord)q.coord=state.jobs[match].coord;return q;}
      if(coord && !isDelete && cell?.name===q.name && !cell.status && !state.jobs.some(j=>j.name===q.name&&!j.coord)) {q.phase='done';q.reason='予定した施設の建築済み状態を確認';return q;}
      if(coord && isDelete && cell?.empty){q.phase='done';q.reason='対象施設の削除済み状態を確認';return q;}
      if(q.phase==='submitted'||q.phase==='observed'){q.reason='開始済み作業の結果確認待ち。重複送信しません。';return q;}
      if(coord && (!cell || (isDelete?(cell.empty||cell.name!==q.name||cell.status):!cell.empty))) {
        if(q.mode==='auto'&&!isDelete){q.coord=null;q.phase='queued';q.reason='他の操作を優先し、別の空きマスを探します';}
        else {q.phase='hold';q.reason='他の操作または区画変更を優先。指定マスの予定は保留';}
      } else {q.phase='queued';q.reason='実行待ち';}
      return q;
    });
  }
  function discoverAction(doc, base, name, action) {
    if(denied(doc.body.textContent))throw Error('同盟拠点の操作権限がありません。');
    const wanted=action==='delete'?/^(?:この(?:施設|建物)を)?削除(?:する|します|を開始する)$/ : /^この(?:施設|建物)(?:の建設準備をする|を建設します|を建設する)$/;
    const found=[];
    for(const a of doc.querySelectorAll('a[href]')) {
      if(!wanted.test(normalize(a.textContent)))continue;
      const container=a.closest('table');
      if(!container)continue;
      // A facility card must name exactly the intended facility in its heading.
      const headings=Array.from(container.querySelectorAll('th')).map(h=>normalize(h.textContent));
      if(!headings.includes(normalize(name)) && !Array.from(container.querySelectorAll('a')).some(link=>normalize(link.textContent)===`[${normalize(name)}]`))continue;
      const href=a.getAttribute('href');if(!href||/^(?:javascript:|#)/i.test(href)||a.hasAttribute('onclick'))continue;
      found.push(safeURL(href,base).href);
    }
    if(found.length!==1)throw Error(`${name}：${action==='delete'?'削除開始':'建設'}リンクを一意に確認できません。条件未達または画面形式未対応のため保留します。`);
    return found[0];
  }
  function filterURL(doc,base,coord){
    const form=doc.querySelector('form[name="facilityAllForm"]');if(!form||normalize(form.getAttribute('method')||'get')!=='get')return null;
    const url=safeURL(form.getAttribute('action')||base,base);
    if(url.pathname!=='/alliance/facility/select_facility.php')return null;
    const params=new URLSearchParams();for(const input of form.querySelectorAll('input[name]')){
      if(!['fad','x','y'].includes(input.name)||input.disabled)continue;params.set(input.name,input.value);
    }
    if(params.get('fad')!=='1'||`${params.get('x')},${params.get('y')}`!==coord)return null;
    url.search=params.toString();url.hash='';return url.href;
  }
  function conditionReason(doc,name){
    const cards=Array.from(doc.querySelectorAll('table')).filter(table=>Array.from(table.querySelectorAll('th')).some(th=>normalize(th.textContent)===normalize(name)));
    if(cards.length!==1)return null;
    const card=cards[0],status=card.querySelector('td.center')?.textContent||card.textContent;
    if(/必要な同盟員の人数を満たしていません/.test(status))return '必要な同盟員の人数を満たしていません';
    if(/建設可能上限に達/.test(status))return '建設上限に達しています';
    const required=status.match(/次の施設が必要です\s*([^\n]+)/);
    if(required)return required[1].trim()+'の完成待ち';
    return null;
  }
  function checkPermission(doc,base,names){
    if(denied(doc.body.textContent||''))return 'denied';
    for(const name of names)for(const action of ['build','delete']){
      try{discoverAction(doc,base,name,action);return 'allowed';}catch{}
    }
    return 'error';
  }
  class Runner {
    constructor(options){Object.assign(this,options);this.running=false;this.controller=null;this.history=[];this.runId=globalThis.crypto?.randomUUID?.()||String(Date.now());}
    event(type,data={}){const entry={time:Date.now(),runId:this.runId,type,...data};this.history.push(entry);this.history=this.history.slice(-150);try{this.onEvent?.(entry);}catch{}return entry;}
    async page(url,{mutation=false}={}){
      this.event(mutation?'send-request':'read-request',{path:new URL(url).pathname});
      const response=await fetch(url,{credentials:'same-origin',cache:'no-store',signal:this.controller?.signal,redirect:mutation?'follow':'error'});
      this.event(mutation?'send-response':'read-response',{status:response.status,path:response.url?new URL(response.url).pathname:null});
      if(!response.ok)throw Error(`画面取得に失敗しました（${response.status}）。`);
      if(response.url)safeURL(response.url,this.base);
      const bytes=await response.arrayBuffer();
      const prefix=new TextDecoder('windows-1252').decode(bytes.slice(0,4096));
      const charset=response.headers.get('content-type')?.match(/charset\s*=\s*["']?([\w-]+)/i)?.[1] || prefix.match(/charset\s*=\s*["']?([\w-]+)/i)?.[1] || 'utf-8';
      const text=new TextDecoder(charset).decode(bytes);return new DOMParser().parseFromString(text,'text/html');
    }
    async renderedVillage(){
      // Read-only navigation: allow the game's own scripts to finish rendering its map.
      return new Promise((resolve,reject)=>{
        const frame=document.createElement('iframe');frame.hidden=true;frame.setAttribute('aria-hidden','true');
        let poll,finished=false;
        const finish=(error,doc)=>{if(finished)return;finished=true;clearInterval(poll);clearTimeout(timeout);this.controller?.signal.removeEventListener('abort',abort);frame.remove();error?reject(error):resolve(doc);};
        const abort=()=>finish(new DOMException('停止しました','AbortError'));
        const timeout=setTimeout(()=>finish(Error('画面の読み込み後も区画情報を取得できません。「読み取り情報を保存」を利用してください。')),15000);
        frame.onload=()=>{
          const check=()=>{try{
            const live=frame.contentDocument;if(!live?.body)return;
            if(denied(live.body.textContent)){finish(Error('同盟拠点の操作権限がありません。'));return;}
            // Require two identical readable snapshots; avoid a half-rendered board.
            const snapshot=new DOMParser().parseFromString(live.documentElement.outerHTML,'text/html');
            this.lastDocument=snapshot;
            let state;try{state=parseVillage(snapshot,this.base,this.names);}catch{return;}
            if(this.frameFingerprint===state.fingerprint){finish(null,snapshot);}else this.frameFingerprint=state.fingerprint;
          }catch(error){finish(error);}};
          this.frameFingerprint=null;poll=setInterval(check,500);check();
        };
        frame.onerror=()=>finish(Error('同盟拠点画面の読み込みに失敗しました。'));
        if(this.controller?.signal.aborted){abort();return;}
        this.controller?.signal.addEventListener('abort',abort,{once:true});
        frame.src=this.base+'/alliance/village.php';document.body.append(frame);
      });
    }
    async refresh(){
      let doc=await this.page(this.base+'/alliance/village.php');this.lastDocument=doc;
      let state;
      try{state=parseVillage(doc,this.base,this.names);}catch(error){
        this.lastError={message:error.message,trace:error.diagnostics||null};
        if(!error.message.startsWith('区画リンクを読み取れません'))throw error;
        this.onMessage('区画情報を確認するため、同盟拠点画面の読み込み完了を待っています。');
        doc=await this.renderedVillage();state=parseVillage(doc,this.base,this.names);
      }
      this.lastError=null;this.onState(state);return state;
    }
    async permission(state){
      const full=state.jobsKnown&&state.jobs.some(job=>job.kind==='prepare');
      const target=[...state.cells.values()].find(cell=>cell.empty)||[...state.cells.values()].find(cell=>/\/facility\/facility\.php(?:\?|$)/.test(cell.url));
      if(!target)return full?'capacity':'error';
      const url=safeURL(target.url,this.base);
      if(!/\/facility\/(?:select_facility|facility)\.php$/.test(url.pathname))return 'error';
      const doc=await this.page(url.href),result=checkPermission(doc,url.href,this.names);
      // No action link while the reservation is full says nothing about role authority.
      return result==='error'&&full?'capacity':result;
    }
    suspend(reason='ページ移動'){this.generation=(this.generation||0)+1;this.running=false;clearTimeout(this.timer);this.controller?.abort();this.event('suspended',{reason,nextCheckAt:this.nextCheckAt||null});}
    stop(reason='手動停止'){this.suspend(reason);this.nextCheckAt=null;this.event('stopped',{reason});this.onMessage('停止しました（'+reason+'）。送信済みの作業は取り消しません。');}
    arm(at){this.nextCheckAt=at;const callback=()=>{this.event('timer-fired',{plannedAt:this.nextCheckAt});return this.tick();};const delay=Math.min(2147483647,Math.max(1000,at-Date.now()));this.timer=this.schedule?this.schedule(callback,delay):setTimeout(callback,delay);this.event('scheduled',{nextCheckAt:at,delay});}
    start({at=null}={}){if(this.running)return;if(!navigator.locks){this.event('start-failed',{reason:'Web Locks未対応'});this.onMessage('このブラウザでは複数タブの重複実行防止に対応できません。');return;}this.running=true;this.lastRunError=null;this.event('started');if(Number.isFinite(at)&&at>Date.now())this.arm(at);else this.tick();}
    async tick(){
      if(!this.running)return;
      const generation=this.generation||0;
      if(this.canRun&&!this.canRun()){this.suspend('別のタブへ実行を引き継ぎ');return;}
      this.nextCheckAt=null;this.event('tick-start');
      const wakeTimes=[];
      this.controller=new AbortController();
      try {
        await navigator.locks.request('alliance-builder:'+this.base,{ifAvailable:true},async lock=>{
          if(!lock){this.onMessage('別タブの操作完了を待っています。');return;}
          if(!this.running||generation!==(this.generation||0)||(this.canRun&&!this.canRun()))return;
          let state=await this.refresh();
          if(!this.running||generation!==(this.generation||0))return;
          let queue=reconcile(this.getQueue(),state);this.setQueue(queue);
          this.event('state-read',{jobs:state.jobs.map(j=>({kind:j.kind,name:j.name,coord:j.coord,finishAt:j.finishAt})),queue:queue.map(q=>({action:q.action,name:q.name,coord:q.coord,phase:q.phase}))});
          if(!state.jobsKnown)throw Error('実行中の作業欄を確認できません。重複操作を防ぐため停止しました。');
          // One transmission per tick. Deletion has its own lane; never cancel existing jobs.
          for(const action of ['delete','build']){
            const q=queue.find(q=>(q.action==='delete'?'delete':'build')===action&&!['done','observed','submitted'].includes(q.phase));
            if(!q||q.phase==='hold')continue;
            const wait=(reason,jobs=[])=>{
              const times=jobs.map(job=>job.finishAt).filter(time=>Number.isFinite(time)&&time+2000>Date.now());
              const wake=times.length?Math.min(...times)+2000:Date.now()+30000;wakeTimes.push(wake);
              q.phase='waiting';q.reason=reason+(times.length?'（'+new Date(wake).toLocaleString('ja-JP',{timeZone:'Asia/Tokyo',hour12:false})+' に再確認）':'（完成時刻を取得できないため30秒後に再確認）');
              this.event('waiting',{name:q.name,coord:q.coord,reason,nextCheckAt:wake});
              this.setQueue(queue);this.onMessage(q.name+'：'+q.reason);
            };
            if(state.jobs.some(j=>action==='delete'?j.kind==='delete':j.kind==='prepare')){wait(action==='delete'?'他の削除作業の完了待ち':'建設準備枠が空くまで待機',state.jobs.filter(j=>action==='delete'?j.kind==='delete':j.kind==='build'));continue;}
            if(action==='build'){
              const facility=this.facilities?.find(f=>f[0]===q.name);
              if(facility){const availability=facilityAvailability(facility,state);if(availability.status==='blocked'){wait(availability.reasons.map(reason=>reason.startsWith('前提施設未完成：')?reason.slice('前提施設未完成：'.length)+'の完成待ち':reason).join(' ／ '),availability.reasons.every(reason=>reason.startsWith('前提施設未完成：'))?state.jobs.filter(job=>job.name===facility[4]&&job.kind!=='delete'):[]);continue;}}
            }
            if(action==='build'&&state.jobs.some(j=>j.kind!=='delete'&&!j.coord&&j.name===q.name)){this.onMessage('同名施設の他の作業を優先して待機しています。');continue;}
            let coord=q.coord;
            if(action==='build'&&q.mode==='auto'&&!coord){const reserved=new Set(queue.filter(other=>other!==q&&other.mode==='manual'&&other.phase!=='done').map(other=>other.coord));coord=[...state.cells.values()].filter(c=>c.empty&&!reserved.has(c.coord)).sort((a,b)=>a.coord.localeCompare(b.coord))[0]?.coord;}
            if(!coord){this.onMessage('空きマスがないため待機しています。');continue;}
            const cell=state.cells.get(coord);if(!cell)continue;
            if(/(?:delete|destroy|remove|cancel|create|build)(?:[_.\/=?&-]|$)/i.test(cell.url))throw Error('施設詳細画面への安全なリンクを確認できません。');
            let detail=await this.page(cell.url),detailURL=cell.url;
            let actionURL;try{actionURL=discoverAction(detail,detailURL,q.name,action);}catch(error){
              if(denied(detail.body.textContent)){this.onPermission?.('denied');throw error;}
              const filtered=action==='build'?filterURL(detail,detailURL,coord):null;
              if(filtered){detail=await this.page(filtered);detailURL=filtered;
                if(denied(detail.body.textContent)){this.onPermission?.('denied');throw Error('同盟拠点の操作権限がありません。');}
                try{actionURL=discoverAction(detail,detailURL,q.name,action);}catch{const reason=conditionReason(detail,q.name);if(reason){wait(reason);continue;}}
              }
              if(!actionURL){const reason=conditionReason(detail,q.name);if(reason){wait(reason);continue;}q.phase='hold';q.reason='施設の実行リンクを読めません。条件未達とは区別して保留します。';this.setQueue(queue);this.onMessage(q.name+'：'+q.reason);continue;}
            }
            this.onPermission?.('allowed');
            const actionCoord=coordOf(new URL(actionURL));if(actionCoord!==coord)throw Error('実行リンクの座標が予定と一致しないため停止しました。');
            // Refetch board state immediately before transmission.
            const latest=await this.refresh();
            if(latest.fingerprint!==state.fingerprint){this.setQueue(reconcile(queue,latest));this.onMessage('他の操作を検出しました。最新状態を優先し、次回再確認します。');return;}
            if(!this.running||generation!==(this.generation||0)||(this.canRun&&!this.canRun()))return;
            q.coord=coord;q.phase='submitted';q.reason='送信前に記録済み。結果確認まで再送しません。';this.setQueue(queue);
            this.event('before-send',{name:q.name,coord,action});
            // No automatic retry: even a lost response may mean the server accepted the action.
            const result=await this.page(actionURL,{mutation:true});
            if(denied(result.body.textContent))throw Error('操作権限エラー。送信結果を確認してください。');
            this.onMessage(`${q.name} (${coord})の${action==='delete'?'削除':'建設'}を送信しました。結果を確認しています。`);
            state=await this.refresh();this.setQueue(reconcile(queue,state));return;
          }
        });
      } catch(error){if(error.name!=='AbortError'&&generation===(this.generation||0)){this.running=false;this.lastRunError={time:Date.now(),message:error.message};this.event('failed',{reason:error.message});this.onMessage(error.message);}}
      finally {if(this.running&&generation===(this.generation||0))this.arm(wakeTimes.length?Math.min(...wakeTimes):Date.now()+30000);}
    }
  }
  class RunControl {
    constructor({storage,key,runner,owner}){Object.assign(this,{storage,key,runner,owner});runner.canRun=()=>{const state=this.read();return state.enabled&&state.owner===this.owner;};}
    read(){const state=JSON.parse(this.storage.getItem(this.key)||'null');if(!state||state.enabled!==true)return {enabled:false,owner:null,wakeAt:null};return {enabled:true,owner:state.owner,wakeAt:Number.isFinite(state.wakeAt)?state.wakeAt:null};}
    write(state){this.storage.setItem(this.key,serialize({...state,updatedAt:Date.now()}));}
    start(){this.write({enabled:true,owner:this.owner,wakeAt:null});this.runner.start();}
    resume(){const state=this.read();if(!state.enabled)return false;this.write({...state,owner:this.owner});this.runner.start({at:state.wakeAt});return true;}
    stop(){this.write({enabled:false,owner:this.owner,wakeAt:null});this.runner.stop();}
    suspend(){this.runner.suspend('ページ移動・再読み込み（自動再開を予約）');}
    onEvent(event){const state=this.read();if(state.owner!==this.owner)return;
      if(event.type==='scheduled')this.write({...state,wakeAt:event.nextCheckAt});
      if(['stopped','failed','start-failed'].includes(event.type))this.write({enabled:false,owner:this.owner,wakeAt:null});
    }
    changed(){const state=this.read();if(this.runner.running&&(!state.enabled||state.owner!==this.owner))this.runner.suspend(state.enabled?'別のゲームタブへ実行を引き継ぎ':'停止操作を反映');}
  }
  const api={RunControl,jobFinish,filterURL,conditionReason,checkPermission,facilityAvailability,serialize,normalize,coordOf,safeURL,parseVillage,reconcile,discoverAction,Runner};
  if(typeof module!=='undefined'&&module.exports)module.exports=api;else scope.AllianceBuilderCore=api;
})(globalThis);



(() => {
  'use strict';
  if(window.top!==window.self)return;
  if (document.getElementById('ab-planner')) return;
  const serialize = globalThis.AllianceBuilderCore?.serialize || JSON.stringify;
  const demo = location.protocol === 'file:' || !location.hostname.endsWith('.3gokushi.jp');
  // Only explicit game messages can establish a denial. Link presence does not prove authority.
  const sourceText = document.body.textContent || '';
  let permission = demo ? 'allowed' : /(?:役職|権限)[^。\n]{0,60}(?:ありません|ないため|必要|できません|出来ません)|(?:操作|管理|建設)[^。\n]{0,30}権限[^。\n]{0,30}(?:ありません|必要)/.test(sourceText) ? 'denied' : 'unknown';
  const facilities = [
    ['大禁衛府',1,2,6,''],['鉄材研究所',2,5,3,''],['石材研究所',4,5,3,'鉄材研究所'],
    ['木材研究所',6,5,3,'石材研究所'],['食糧研究所',8,5,3,'木材研究所'],
    ['大練兵所',2,5,3,''],['大兵舎',4,5,3,'大練兵所'],['大弓兵舎',6,5,3,'大兵舎'],
    ['大厩舎',8,5,3,'大弓兵舎'],['大兵器工房',10,5,3,'大厩舎'],
    ['大斧兵舎',12,5,3,'大兵器工房'],['大双兵舎',12,5,3,'大兵器工房'],['大錘兵舎',12,5,3,'大兵器工房'],
    ['大重装兵舎',18,2,6,''],['祈祷所',2,5,6,''],['軍令台・防御',6,5,6,'祈祷所'],['軍令台・攻撃',8,5,6,'軍令台・防御']
  ];
  const available = new Set();
  const occupied = new Map();
  const inProgress = new Map();
  if (!demo) document.querySelectorAll('a[href],area[href]').forEach(a => {
    try {
      const u = new URL(a.getAttribute('href'), location.href);
      if (u.origin !== location.origin || !u.pathname.startsWith('/alliance/facility/')) return;
      const x = u.searchParams.get('x'), y = u.searchParams.get('y');
      if (!/^[0-6]$/.test(x) || !/^[0-6]$/.test(y)) return;
      const key = `${x},${y}`;
      const description = [a.textContent, a.title, a.getAttribute('alt'), a.getAttribute('aria-label'), ...Array.from(a.querySelectorAll('img')).flatMap(img=>[img.alt,img.title])].filter(Boolean).join(' ');
      const facility = facilities.find(f=>description.replace(/\s/g,'').includes(f[0]));
      if (facility) occupied.set(key, facility[0]);
      else if (u.pathname === '/alliance/facility/select_facility.php') available.add(key);
      else occupied.set(key, '施設名未取得');
    } catch {}
  });
  // Preview coordinates are illustrative; the live UI only uses discovered links.
  if (demo) for(let x=0;x<7;x++) for(let y=0;y<7;y++) if (!(x < 2 && y >= 2 && y <= 4)) available.add(`${x},${y}`);
  if (demo) {
    // Illustrative fixtures only: these are not the user's server data.
    occupied.set('2,0','鉄材研究所');
    occupied.set('3,0','石材研究所');
    occupied.set('6,6','祈祷所');
    occupied.set('2,1','軍令台・防御');
    occupied.set('3,1','大兵舎');
    inProgress.set('6,6','削除中');
    inProgress.set('2,1','建設中');
    inProgress.set('3,1','建設準備中');
  }
  occupied.forEach((_,key)=>available.delete(key));
  const storageKey = `alliance-builder-ui-v1:${location.host || 'preview'}`;
  let queue = [];
  try {
    let saved = JSON.parse(localStorage.getItem(storageKey) || '[]');
    if(typeof saved==='string')saved=JSON.parse(saved);
    if (Array.isArray(saved)) queue = saved.filter(q => q && typeof q.name==='string' && (q.action==='delete' ? q.mode==='manual' && /^[0-6],[0-6]$/.test(q.coord) : facilities.some(f=>f[0]===q.name) && (q.mode==='auto' || (q.mode==='manual' && /^[0-6],[0-6]$/.test(q.coord))))).map(q=>({...q,action:q.action==='delete'?'delete':'build',name:q.name,mode:q.mode,coord:/^[0-6],[0-6]$/.test(q.coord)?q.coord:null}));
  } catch {}
  let selected = null;
  let availabilityState = demo ? {text:'同盟人数 1 / 82',cells:new Map([...available].map(coord=>[coord,{coord,empty:true}])),jobs:[]} : null;
  if(demo)occupied.forEach((name,coord)=>availabilityState.cells.set(coord,{coord,name,empty:false,status:inProgress.get(coord)||null}));
  const style = document.createElement('style');
  style.textContent = `
  #ab-planner{--ink:#24362e;--green:#245b42;--muted:#728076;box-sizing:border-box;color:var(--ink);font:14px/1.6 system-ui,sans-serif;background:#f5f6f0;border:1px solid #d8dfd3;border-radius:18px;box-shadow:0 18px 70px #162c2333;position:fixed;inset:4vh 3vw;z-index:99999;overflow:auto;text-align:left}
  #ab-planner *{box-sizing:border-box}#ab-planner button,#ab-planner select{font:inherit}#ab-planner button{cursor:pointer}#ab-planner button:disabled{cursor:not-allowed;opacity:.4}
  #ab-planner header{display:flex;align-items:center;justify-content:space-between;padding:24px 30px;background:#fff;border-bottom:1px solid #e1e6dd}#ab-planner h1{font-size:23px;margin:0}#ab-planner h2{font-size:17px;margin:0 0 16px}#ab-planner p{margin:6px 0}#ab-planner small{color:var(--muted)}
  #ab-planner .ab-tag{display:inline-block;color:#396149;background:#edf3e8;padding:3px 10px;border-radius:20px;font-size:12px}#ab-planner .ab-body{padding:24px 30px}#ab-planner .ab-notice{border-left:3px solid #b39751;background:#fff9e9;padding:10px 14px;margin-bottom:22px;font-size:13px}
  #ab-planner .ab-layout{display:grid;grid-template-columns:minmax(300px,1fr) minmax(360px,1.3fr);gap:22px}#ab-planner .ab-card{background:#fff;border:1px solid #e0e5db;border-radius:12px;padding:22px}#ab-planner label.ab-label{display:block;font-weight:600;margin:16px 0 6px}#ab-planner select{width:100%;padding:11px;border:1px solid #cbd5c8;border-radius:7px;background:#fff;color:var(--ink)}
  #ab-planner .ab-facts{display:grid;grid-template-columns:repeat(3,1fr);gap:8px;background:#f6f7f2;padding:13px;margin:12px 0;border-radius:8px}#ab-planner .ab-facts span{display:block;font-size:12px;color:var(--muted)}#ab-planner .ab-choice{display:block;padding:12px;border:1px solid #dbe2d6;border-radius:8px;margin:8px 0}#ab-planner input{accent-color:var(--green)}
  #ab-planner .ab-btn{padding:9px 14px;border:1px solid #ced8ca;border-radius:7px;background:white;color:var(--ink)}#ab-planner .ab-primary{background:var(--green);color:white;border-color:var(--green);width:100%;margin-top:18px;padding:12px}#ab-planner .ab-mapwrap{height:325px;display:flex;align-items:center;justify-content:center;overflow:hidden}#ab-planner .ab-map{display:grid;grid-template-columns:repeat(7,31px);gap:4px;transform:rotate(45deg)}
  #ab-planner .ab-cell{height:31px;border:1px solid #c8d7c0;border-radius:3px;background:#ecf1e7;color:#526b4c;padding:0;font-size:9px}#ab-planner .ab-cell span{display:block;transform:rotate(-45deg)}#ab-planner .ab-cell:disabled{background:#151515;border-color:#151515;opacity:1;color:#fff}#ab-planner .ab-cell.ab-selected{background:#245b42;color:white;border:2px solid #c5a35a}#ab-planner .ab-cell.ab-reserved{background:#ead9b0;color:#66512b}
  #ab-planner .ab-listcard{margin-top:22px}#ab-planner .ab-row{display:flex;align-items:center;gap:10px;border-top:1px solid #e6eade;padding:12px 0}#ab-planner .ab-row strong{flex:1}#ab-planner .ab-num{color:#819079;width:24px}#ab-planner .ab-empty{text-align:center;padding:32px;color:var(--muted)}#ab-planner .ab-toolbar{display:flex;justify-content:space-between;align-items:center;gap:12px}#ab-planner .ab-message{min-height:25px;color:#866329;margin-top:9px}#ab-planner .ab-mini{padding:3px 9px}#ab-open{position:fixed;bottom:20px;right:20px;z-index:99998;background:#245b42;color:white;border:0;border-radius:25px;padding:13px 22px;cursor:pointer}
  @media(max-width:800px){#ab-planner{inset:8px}#ab-planner .ab-layout{grid-template-columns:1fr}#ab-planner .ab-body{padding:16px}#ab-planner header{padding:16px}#ab-planner .ab-row{flex-wrap:wrap}}
  `;
  style.textContent += `
  #ab-planner .ab-mapwrap{height:450px;overflow:auto}
  #ab-planner .ab-map{grid-template-columns:repeat(7,40px);flex-shrink:0}
  #ab-planner .ab-cell{height:40px}
  #ab-planner .ab-cell span{font-size:9px;line-height:1.2}
  #ab-planner .ab-cell.ab-occupied{background:#d9e5f0;border-color:#809bb3;color:#213e56;opacity:1}
  #ab-planner .ab-cell.ab-occupied span{font-size:8px;overflow-wrap:anywhere}
  #ab-planner .ab-cell em{display:block;font-size:8px;font-style:normal;margin-top:2px}
  `;
  document.head.append(style);
  const root = document.createElement('section');root.id='ab-planner';
  root.innerHTML = `<header><div><span class="ab-tag">${demo?'プレビュー':location.hostname} · UI試作</span><h1>同盟拠点 建設プランナー</h1><small>施設を選んで、建てる順番を準備しましょう。</small></div><button class="ab-btn" id="ab-close" aria-label="閉じる">閉じる</button></header>
  <div class="ab-body"><div class="ab-notice">この版は建設予定を編集するUIです。ゲーム内の建設・建設準備は実行しません。${demo?'黒い6マスは画像から推定した配置です（x=0〜1、y=2〜4）。実画面では区画リンクから選択候補を取得します。':''}</div>
  <div class="ab-layout"><div class="ab-card"><h2>01　施設を選ぶ</h2><label class="ab-label" for="ab-facility">建設する施設</label><select id="ab-facility"></select><div id="ab-details"></div>
  <label class="ab-label">配置方法</label><label class="ab-choice"><input type="radio" name="ab-mode" value="auto" checked> 自動配置<br><small>空いている建築可能マスを自動で選びます。</small></label><label class="ab-choice"><input type="radio" name="ab-mode" value="manual"> マス指定<br><small>右の配置図から建設場所を選びます。</small></label>
  <button id="ab-add" class="ab-btn ab-primary">建設予定に追加</button><div id="ab-message" class="ab-message" role="status"></div></div>
  <div class="ab-card"><div class="ab-toolbar"><h2>02　建設場所</h2><span class="ab-tag" id="ab-position">自動配置</span></div><small>奥 (0,0) ／ 左 (0,6) ／ 右 (6,0) ／ 手前 (6,6)</small><div class="ab-mapwrap"><div class="ab-map" id="ab-map"></div></div><p><small>緑：選択候補　濃緑：選択中　黄：予定あり　黒：建築不可・選択対象外</small></p><small>${demo?'49マスのうち、推定した建築不可6マスを黒塗りしています。候補は43マスです。':`施設選択リンクを ${available.size} マス取得しました。リンクのない区画は選択できません。施設ごとの建設条件は未照合です。`}</small></div></div>
  <div class="ab-card ab-listcard"><div class="ab-toolbar"><h2>03　建設予定 <span id="ab-count"></span></h2><span class="ab-tag">建設中1件 ＋ 準備中1件</span></div><div id="ab-queue"></div><p><small>上から順に処理する予定です。前提施設・人数・上限などの実行時判定は、次の実装で追加します。</small></p><div class="ab-toolbar"><small id="ab-save">このサーバーの予定として保存</small><button class="ab-btn" disabled>自動建設を開始（未実装）</button></div></div></div>`;
  document.body.append(root);
  const $ = id=>root.querySelector('#'+id);
  const permissionPanel=document.createElement('div');permissionPanel.id='ab-permission';permissionPanel.setAttribute('role','alert');permissionPanel.style.cssText='padding:16px 18px;border:1px solid #cf8a7d;border-radius:9px;background:#fff0ec;color:#883b2c;margin-bottom:20px';
  root.querySelector('.ab-layout').before(permissionPanel);
  if(demo){
    const controls=document.createElement('label');controls.className='ab-label';controls.textContent='プレビュー：同盟拠点の操作権限';
    const select=document.createElement('select');select.id='ab-permission-demo';
    [['allowed','役職あり（操作可能な表示例）'],['capacity','予約枠が満杯（待機表示例）'],['denied','役職なし（エラー表示例）'],['unknown','権限を確認できない場合']].forEach(([value,label])=>{const option=document.createElement('option');option.value=value;option.textContent=label;select.append(option);});
    select.onchange=()=>{permission=select.value;selected=null;renderQueue();renderMap();renderPermission();};controls.append(select);root.querySelector('.ab-notice').after(controls);
  }
  function renderPermission(){
    permissionPanel.hidden=false;
    permissionPanel.style.background=permission==='denied'?'#fff0ec':'#f0f5fa';
    permissionPanel.style.borderColor=permission==='denied'?'#cf8a7d':'#b4c8da';
    permissionPanel.style.color=permission==='denied'?'#883b2c':'#39566d';
    permissionPanel.setAttribute('role',permission==='denied'?'alert':'status');
    permissionPanel.textContent=permission==='allowed'?'操作権限あり：ゲーム画面の建設・削除操作リンクを確認しました。':permission==='capacity'?'建設予約枠が満杯のため待機中です。権限エラーではありません。枠が空いた後にゲーム側の操作可否を確認します。':permission==='denied'?'操作権限なし：ゲーム側の権限エラーを確認しました。':permission==='error'?'権限確認エラー：権限を確定できる情報を取得できませんでした。':'操作権限を確認中です。「最新状態を取得」でゲーム側の操作画面を確認します。';
    if(permission==='allowed'){permissionPanel.style.background='#eef5ee';permissionPanel.style.borderColor='#a7c5ac';permissionPanel.style.color='#285b3d';}
    if(permission==='error'){permissionPanel.style.background='#fff0ec';permissionPanel.style.borderColor='#cf8a7d';permissionPanel.style.color='#883b2c';}
    if($('ab-run-start'))$('ab-run-start').disabled=demo||!['allowed','capacity'].includes(permission);
    $('ab-add').disabled=permission==='denied';
    if($('ab-delete-add'))$('ab-delete-add').disabled=permission==='denied'||!$('ab-delete-target').value;
    ['ab-facility','ab-delete-target'].forEach(id=>{if($(id))$(id).disabled=permission==='denied';});
    root.querySelectorAll('input[name="ab-mode"]').forEach(input=>input.disabled=permission==='denied');
  }
  const statusPanel=document.createElement('section');statusPanel.className='ab-card';statusPanel.style.marginBottom='22px';
  const statusTitle=document.createElement('h2');statusTitle.textContent='現在の作業';statusPanel.append(statusTitle);
  const statusNote=document.createElement('small');statusNote.textContent=demo?'表示例です。施設・残り時間・座標は実サーバーの現在値ではありません。':'最新状態の取得待ちです。未取得を空き状態とは判定しません。';statusPanel.append(statusNote);
  const statusGrid=document.createElement('div');statusGrid.className='ab-status-grid';
  [['建設中','軍令台・防御','残り 02:15:23'],['建設準備中','大兵舎','完成まで 05:15:33'],['削除中','祈祷所','残り 00:59:45']].forEach(([title,name,time])=>{
    const card=document.createElement('div');card.className='ab-status-slot';
    if(title==='削除中')card.style.borderTopColor='#b86b5a';
    const heading=document.createElement('small');heading.textContent=title;
    const facility=document.createElement('strong');facility.textContent=demo?name:'未取得';
    const remaining=document.createElement('div');remaining.textContent=demo?time:'ゲーム画面で確認してください';
    card.append(heading,facility,remaining);statusGrid.append(card);
  });
  statusPanel.append(statusGrid);const parallelNote=document.createElement('p');parallelNote.textContent='建設と削除は並行して進みます。削除時間は1時間。削除が完了するまで、そのマスは建設対象にしません。';parallelNote.style.fontSize='12px';statusPanel.append(parallelNote);
  root.querySelector('.ab-layout').before(statusPanel);
  style.textContent += `#ab-planner .ab-status-grid{display:grid;grid-template-columns:repeat(3,minmax(0,1fr));gap:12px;margin-top:14px}#ab-planner .ab-status-slot{background:#f6f7f2;padding:14px;border-radius:8px;border-top:3px solid #537c60}#ab-planner .ab-status-slot strong{display:block;font-size:17px;margin:6px 0}#ab-planner .ab-queue-group{margin-top:20px}#ab-planner .ab-queue-group h3{font-size:15px;margin-bottom:8px}@media(max-width:600px){#ab-planner .ab-status-grid{grid-template-columns:1fr}}`;
  root.querySelector('.ab-listcard h2').firstChild.textContent='03　作業予定 ';
  root.querySelector('.ab-listcard .ab-tag').textContent='建設と削除は別の予定';
  root.querySelector('.ab-listcard p small').textContent='各リストの上から順に処理する予定です。実際の建設・削除の実行と進行状況の取得は未実装です。削除を同時に実行できる件数は未確認です。';
  root.querySelector('.ab-notice').firstChild.textContent='この版は作業予定を編集するUIです。ゲーム内の建設・建設準備・施設削除は実行しません。';
  const deletePanel = document.createElement('div');
  deletePanel.innerHTML = `<hr style="border:0;border-top:1px solid #e0e5db;margin:22px 0"><h2>施設の削除</h2><small>建築済みの施設を選び、削除予定を追加します。削除時間は1時間です。</small><label class="ab-label" for="ab-delete-target">削除する施設</label><select id="ab-delete-target"></select><button id="ab-delete-add" class="ab-btn" style="width:100%;margin-top:12px;color:#9a3e31">削除予定に追加（1時間）</button>`;
  $('ab-add').parentElement.append(deletePanel);
  function refreshDeleteTargets(){
    const select=$('ab-delete-target');const previous=select.value;select.replaceChildren();
    occupied.forEach((name,coord)=>{
      const option=document.createElement('option');option.value=coord;
      const queued=queue.some(q=>q.action==='delete'&&q.coord===coord), active=inProgress.get(coord);
      option.textContent=`${name} (${coord})${active?` — ${active}`:queued?' — 削除予定あり':''}`;option.disabled=queued||!!active;select.append(option);
    });
    if(!occupied.size){const option=document.createElement('option');option.textContent='建築済み施設が取得されていません';option.value='';select.append(option);}
    if(Array.from(select.options).some(o=>o.value===previous&&!o.disabled))select.value=previous;
    else select.selectedIndex=Array.from(select.options).findIndex(o=>!o.disabled);
    $('ab-delete-add').disabled=permission==='denied'||!select.value;
  }
  $('ab-delete-add').onclick=()=>{
    if(permission==='denied'){renderPermission();return;}
    const coord=$('ab-delete-target').value,name=occupied.get(coord);
    if(!name||inProgress.has(coord)||queue.some(q=>q.action==='delete'&&q.coord===coord))return;
    queue.push({action:'delete',name,mode:'manual',coord});save();renderQueue();renderMap();
    $('ab-message').textContent=`${name} (${coord})を削除予定に追加しました。削除時間は1時間です。`;
  };
  const mapNotes = root.querySelector('.ab-mapwrap').nextElementSibling;
  mapNotes.textContent = '緑：空きマス　青：建築済み　黄：建設中・準備中・予定あり　赤：削除中・削除予定　黒：建築不可・選択対象外';
  mapNotes.style.fontSize = '12px';
  if (demo) mapNotes.nextElementSibling.textContent = `施設と作業状態は仮配置です（実サーバーの配置ではありません）。黒6マスは推定位置、空き候補は${available.size}マスです。`;
  else mapNotes.nextElementSibling.textContent = `空き候補${available.size}マス、施設区画${occupied.size}マスを取得。施設名が読み取れない区画は「施設名未取得」と表示します。実画面との照合は未実施です。`;
  const opener=document.createElement('button');opener.id='ab-open';opener.textContent='同盟拠点プランナー';opener.hidden=true;document.body.append(opener);
  $('ab-close').onclick=()=>{root.hidden=true;opener.hidden=false;};opener.onclick=()=>{root.hidden=false;opener.hidden=true;};
  facilities.forEach(f=>{const o=document.createElement('option');o.value=f[0];o.textContent=f[0];$('ab-facility').append(o);});
  function details(){
    const colors={blocked:['#fff0ed','#a43b2c'],candidate:['#eef5ee','#285b3d'],unknown:['#f3f4f5','#66717a']};
    const evaluate=f=>globalThis.AllianceBuilderCore?.facilityAvailability(f,availabilityState)||{status:'unknown',reasons:['判定処理を読み込めません']};
    for(const option of $('ab-facility').options){const item=facilities.find(f=>f[0]===option.value),result=evaluate(item);option.textContent=item[0]+' — '+({blocked:'建築不可',candidate:'基本条件OK',unknown:'未確認'})[result.status];option.style.backgroundColor=colors[result.status][0];option.style.color=colors[result.status][1];}
    const f=facilities.find(f=>f[0]===$('ab-facility').value),result=evaluate(f),[background,color]=colors[result.status];
    $('ab-facility').style.backgroundColor=background;$('ab-facility').style.color=color;$('ab-facility').style.borderColor=color;
    $('ab-details').innerHTML=`<div class="ab-facts"><div><span>必要人数</span>${f[1]}人</div><div><span>建設上限</span>${f[2]}件</div><div><span>基本時間</span>${f[3]}時間</div></div><small>前提施設：${f[4]||'提供画面では表示なし'}</small>`;
    const badge=document.createElement('div');badge.style.cssText='padding:10px 12px;border-radius:7px;margin-top:10px';badge.style.backgroundColor=background;badge.style.color=color;
    const heading=document.createElement('strong');heading.textContent=({blocked:'現在は建築できません',candidate:'基本条件を満たしています（実行時に再確認）',unknown:'建築可否は未確認'})[result.status];badge.append(heading);
    result.reasons.forEach(reason=>{const line=document.createElement('div');line.textContent=reason;badge.append(line);});$('ab-details').append(badge);
    const note=document.createElement('small');note.textContent=(demo?'サンプルの同盟人数は1人です。':'')+'赤：現在建築不可 ／ 緑：基本条件OK ／ 灰：未確認。建築不可でも将来の予定に追加できます。';$('ab-details').append(note);
  }
  function mode(){return root.querySelector('input[name="ab-mode"]:checked').value;}
  function renderMap(){
    $('ab-map').replaceChildren();
    for(let y=0;y<7;y++)for(let x=0;x<7;x++){
      const key=`${x},${y}`, reserved=queue.some(q=>q.mode==='manual'&&q.coord===key);
      const deleting=queue.some(q=>q.action==='delete'&&q.coord===key);
      const building = occupied.get(key);
      const b=document.createElement('button');b.type='button';b.className='ab-cell'+(selected===key?' ab-selected':'')+(reserved?' ab-reserved':'')+(building?' ab-occupied':'');
      const label=document.createElement('span');label.textContent=building || key;
      if(building){const coord=document.createElement('em');coord.textContent=key;label.append(coord);}
      if(deleting){b.style.background='#f5d9d3';b.style.borderColor='#b86b5a';const state=document.createElement('em');state.textContent='削除予定';label.append(state);}
      const active=inProgress.get(key);
      if(active){b.style.background=active==='削除中'?'#edb8ae':'#f1deb2';b.style.borderColor=active==='削除中'?'#a64b39':'#ad9149';const state=document.createElement('em');state.textContent=active;label.append(state);}
      b.append(label);b.title=`座標 (${key})${building?`：${building}${deleting?'（削除予定・1時間）':''}`:reserved?'：建設予定あり':''}`;b.setAttribute('aria-label',b.title);b.setAttribute('aria-pressed',String(selected===key));b.disabled=permission==='denied'||!available.has(key)||reserved||!!building;
      b.onclick=()=>{selected=key;root.querySelector('input[value="manual"]').checked=true;renderMap();};$('ab-map').append(b);
    }
    $('ab-position').textContent=mode()==='auto'?'自動配置':selected?`選択 (${selected})`:'マスを選択してください';
  }
  function save(){try{localStorage.setItem(storageKey,serialize(queue));$('ab-save').textContent='このサーバーの予定として保存済み';}catch{$('ab-save').textContent='保存できませんでした。この画面を閉じると予定が失われます。';}}
  function renderQueue(){
    refreshDeleteTargets();
    $('ab-count').textContent=`(${queue.length}件)`;$('ab-queue').replaceChildren();
    for(const action of ['build','delete']){
      const indices=queue.map((q,i)=>({q,i})).filter(({q})=>(q.action==='delete'?'delete':'build')===action);
      const group=document.createElement('section');group.className='ab-queue-group';
      const heading=document.createElement('h3');heading.textContent=(action==='delete'?'削除予定（各1時間）':'建設予定')+' · '+indices.length+'件';group.append(heading);
      if(!indices.length){const empty=document.createElement('div');empty.className='ab-empty';empty.textContent=action==='delete'?'削除予定はありません。':'建設予定はありません。';group.append(empty);}
      indices.forEach(({q,i},position)=>{
        const row=document.createElement('div');row.className='ab-row';
        const num=document.createElement('span');num.className='ab-num';num.textContent=String(position+1).padStart(2,'0');
        const name=document.createElement('strong');name.textContent=q.name;if(action==='delete')name.style.color='#9a3e31';
        const place=document.createElement('span');place.textContent=q.mode==='auto'?'自動配置':'('+q.coord+')';row.append(num,name,place);
        if(q.phase){const state=document.createElement('small');state.textContent=({queued:'待機',waiting:'条件待ち',hold:'保留',submitted:'送信済み・確認待ち',observed:'進行中',done:'完了'}[q.phase]||q.phase)+'：'+(q.reason||'');row.append(state);}
        if(!['observed','submitted','done'].includes(q.phase)&&q.mode==='manual'&&(action==='delete'?(!occupied.has(q.coord)||inProgress.has(q.coord)):!available.has(q.coord))){const warning=document.createElement('small');warning.textContent='対象マスの状態を再確認してください';row.append(warning);}
        [['↑',-1],['↓',1],['予定から外す',0]].forEach(([label,delta])=>{
          const button=document.createElement('button');button.className='ab-btn ab-mini';button.textContent=label;
          button.setAttribute('aria-label',q.name+' '+(delta===-1?'上へ':delta===1?'下へ':'予定から外す'));
          button.disabled=permission==='denied'||delta!==0&&(position+delta<0||position+delta>=indices.length);
          button.onclick=()=>{if(permission==='denied'){renderPermission();return;}if(delta){const target=indices[position+delta].i;[queue[i],queue[target]]=[queue[target],queue[i]];}else queue.splice(i,1);save();renderQueue();renderMap();};row.append(button);
        });group.append(row);
      });$('ab-queue').append(group);
    }
  }
  $('ab-add').onclick=()=>{
    if(permission==='denied'){renderPermission();return;}
    const m=mode(), name=$('ab-facility').value;
    if(m==='manual'&&(!selected||!available.has(selected)||queue.some(q=>q.coord===selected))){$('ab-message').textContent='建設予定のない候補マスを選んでください。';return;}
    const count=queue.filter(q=>q.action!=='delete'&&q.name===name).length, max=facilities.find(f=>f[0]===name)[2];
    if(count>=max){$('ab-message').textContent=`予定内で${name}の上限${max}件に達しています。`;return;}
    if(queue.filter(q=>q.action!=='delete').length>=available.size){$('ab-message').textContent='候補マス数を超える予定は追加できません。';return;}
    queue.push({action:'build',name,mode:m,coord:m==='manual'?selected:null});selected=null;save();renderQueue();renderMap();$('ab-message').textContent=`${name}を予定に追加しました。`;
  };
  $('ab-facility').onchange=details;root.querySelectorAll('input[name="ab-mode"]').forEach(r=>r.onchange=renderMap);
  const core=globalThis.AllianceBuilderCore;
  const controls=document.createElement('div');controls.className='ab-toolbar';controls.style.marginTop='16px';
  const refresh=document.createElement('button');refresh.className='ab-btn';refresh.textContent='最新状態を取得';
  const start=document.createElement('button');start.id='ab-run-start';start.className='ab-btn';start.textContent='予定の自動実行を開始';
  const stop=document.createElement('button');stop.className='ab-btn';stop.textContent='停止';
  controls.append(refresh,start,stop);statusPanel.append(controls);
  const log=document.createElement('p');log.setAttribute('role','status');statusPanel.append(log);
  const runStatus=document.createElement('p');runStatus.id='ab-run-status';runStatus.setAttribute('role','status');runStatus.textContent='自動実行：停止中（開始ボタンを押してください）';statusTitle.after(runStatus);
  const historyKey=storageKey+':history';
  function recordEvent(entry){
    try{let events=JSON.parse(localStorage.getItem(historyKey)||'[]');if(!Array.isArray(events))events=[];events.push(entry);localStorage.setItem(historyKey,serialize(events.slice(-250)));}catch{}
    if(entry.type==='scheduled')runStatus.textContent='自動実行：待機中 ／ 次回確認 '+new Date(entry.nextCheckAt).toLocaleString('ja-JP',{timeZone:'Asia/Tokyo',hour12:false});
    else if(entry.type==='failed'||entry.type==='start-failed')runStatus.textContent='自動実行：エラーで停止 ／ '+entry.reason;
    else if(entry.type==='stopped')runStatus.textContent='自動実行：停止中 ／ '+entry.reason;
    else if(entry.type==='suspended')runStatus.textContent='自動実行：引き継ぎ待機 ／ '+entry.reason;
    else if(entry.type==='started'||entry.type==='tick-start')runStatus.textContent='自動実行：確認・処理中';
  }
  recordEvent({time:Date.now(),type:'page-loaded',page:location.pathname,version:'0.3.13'});
  const message=text=>{log.textContent=new Date().toLocaleTimeString('ja-JP')+'　'+text;};
  root.querySelector('.ab-listcard .ab-toolbar:last-child button').remove();
  root.querySelector('.ab-notice').firstChild.textContent=demo?'プレビューではゲームへの送信は行いません。':'自動実行は開始ボタンを押した場合のみ動作します。建設・削除は他の役職の操作を優先します。';
  root.querySelector('.ab-listcard p small').textContent='各予定を順に照合します。他の役職による一致した作業は進行を反映し、競合するマス指定は保留します。開始済み作業は取り消しません。';
  function applyState(state){
    availabilityState=state;details();
    available.clear();occupied.clear();inProgress.clear();
    for(const cell of state.cells.values()){
      if(cell.empty)available.add(cell.coord);else occupied.set(cell.coord,cell.name||'施設名未取得');
      if(cell.status)inProgress.set(cell.coord,({build:'建設中',prepare:'建設準備中',delete:'削除中'})[cell.status]);
    }
    state.jobs.forEach(job=>{if(job.coord){inProgress.set(job.coord,({build:'建設中',prepare:'建設準備中',delete:'削除中'})[job.kind]);available.delete(job.coord);}});
    statusNote.textContent='最終取得 '+new Date().toLocaleTimeString('ja-JP')+' ／ '+(state.jobsKnown?'作業欄を確認':'作業欄の判定未対応・実行停止');
    ['build','prepare','delete'].forEach((kind,index)=>{
      const jobs=state.jobs.filter(job=>job.kind===kind), card=statusGrid.children[index];
      card.querySelector('strong').textContent=jobs.length?jobs.map(j=>j.name||'施設名不明').join('、'):state.jobsKnown?'作業なし':'未取得';
      card.lastElementChild.textContent=jobs.map(j=>j.text).join(' / ');
    });
    mapNotes.nextElementSibling.textContent=`空き候補${available.size}マス、施設区画${occupied.size}マス。画像の説明から施設名を取得できない区画は操作を保留します。`;
    renderQueue();renderMap();renderPermission();
  }
  let expectedStorage;
  let control=null;
  const runner=!demo&&core?new core.Runner({base:location.origin,names:facilities.map(f=>f[0]),facilities,
    getQueue:()=>{expectedStorage=localStorage.getItem(storageKey);const stored=JSON.parse(expectedStorage||'[]');if(!Array.isArray(stored))throw Error('予定データを確認できません。');return stored;},
    setQueue:next=>{if(localStorage.getItem(storageKey)!==expectedStorage)throw Error('別の画面で予定が変更されました。停止して再取得します。');queue=next;localStorage.setItem(storageKey,serialize(queue));expectedStorage=localStorage.getItem(storageKey);renderQueue();renderMap();},
    onEvent:event=>{recordEvent(event);control?.onEvent(event);},onState:applyState,onPermission:value=>{permission=value;renderPermission();},onMessage:text=>{if(/権限.*ありません|操作権限エラー/.test(text)){permission='denied';renderPermission();}message(text);}
  }):null;
  const controlKey=storageKey+':run-control';
  if(runner)control=new core.RunControl({storage:localStorage,key:controlKey,runner,owner:runner.runId});
  refresh.disabled=start.disabled=stop.disabled=demo||!core;
  refresh.onclick=async()=>{const active=runner.running;permission='checking';renderPermission();try{const state=await runner.refresh();if(!active&&!runner.running){expectedStorage=localStorage.getItem(storageKey);runner.setQueue(core.reconcile(JSON.parse(expectedStorage||'[]'),state));}permission=await runner.permission(state);renderPermission();message(permission==='allowed'?'最新状態と操作権限ありを確認しました。':permission==='capacity'?'建設予約枠が満杯です。空き枠ができるまで待機します。':permission==='denied'?'操作権限なしを確認しました。':'操作権限の確認に失敗しました。');}catch(error){permission='error';renderPermission();runner.onMessage(error.message);}};
  opener.onclick=()=>{root.hidden=false;opener.hidden=true;if(runner){message('同盟拠点の最新状態を取得しています。');refresh.onclick();}};
  root.querySelector('header .ab-tag').textContent=(demo?'プレビュー':location.hostname)+' · v0.3.13';
  start.onclick=()=>{if(!['allowed','capacity'].includes(permission)){renderPermission();return;}try{control.start();message('自動実行を開始しました。ページ移動・再読み込み後も引き継ぎます。');}catch(error){message('実行状態を保存できないため開始できません：'+error.message);}};
  stop.onclick=()=>{try{control.stop();}catch(error){runner.stop();message(error.message);}};
  const diagnostics=document.createElement('button');diagnostics.className='ab-btn';diagnostics.textContent='読み取り情報を保存';diagnostics.disabled=demo||!core;controls.append(diagnostics);
  diagnostics.onclick=()=>{
    const summarize=doc=>{
      if(!doc)return null;
      const clone=doc.documentElement.cloneNode(true);clone.querySelectorAll('#ab-planner,#ab-open,script,style,noscript').forEach(node=>node.remove());
      const links=Array.from(clone.querySelectorAll('a[href],area[href]')).map(a=>{
        let url;try{const u=new URL(a.getAttribute('href'),location.href);url=u.origin+u.pathname;const params=new URLSearchParams();for(const [key,value]of u.searchParams)params.append(key,/^(x|y)$/.test(key)?value:'[省略]');if(params.size)url+='?'+params;}catch{url='URL形式未対応';}
        return {url,title:a.title,alt:a.getAttribute('alt'),text:a.textContent.trim().slice(0,100),images:Array.from(a.querySelectorAll('img')).map(img=>({alt:img.alt,title:img.title})),hasOnclick:a.hasAttribute('onclick')};
      });
      return {links,text:clone.textContent.replace(/\s+/g,' ').slice(0,30000)};
    };
    let executionHistory=[];try{executionHistory=JSON.parse(localStorage.getItem(historyKey)||'[]');}catch{}
    const data={version:'0.3.13',server:location.host,page:location.pathname,execution:{running:runner.running,control:control?.read(),nextCheckAt:runner.nextCheckAt||null,lastRunError:runner.lastRunError||null,queue,history:executionHistory},lastError:runner.lastError||null,runtime:{urlConstructor:String(URL).slice(0,200),mapConstructor:String(Map).slice(0,200),arrayToJSON:typeof Array.prototype.toJSON},received:summarize(runner.lastDocument),displayed:summarize(document)};
    const blob=new Blob([serialize(data)],{type:'application/json'}),url=URL.createObjectURL(blob),a=document.createElement('a');a.href=url;a.download='alliance-builder-diagnostics.json';a.click();setTimeout(()=>URL.revokeObjectURL(url),1000);
    message('読み取り情報を保存しました。共有前に内容をご確認ください（画面の文字とリンク情報を含みます）。');
  };
  window.addEventListener('storage',event=>{if(event.key===storageKey){const activeOwner=runner?.running&&control?.read().owner===runner.runId;if(activeOwner)runner.stop('別タブで予定変更');try{queue=JSON.parse(event.newValue||'[]');renderQueue();renderMap();}catch{}if(activeOwner)message('別タブで予定が変更されました。自動実行を停止しました。');}});
  window.addEventListener('storage',event=>{if(event.key===controlKey)control?.changed();});
  window.addEventListener('pagehide',()=>control?.suspend());
  window.addEventListener('pageshow',event=>{if(event.persisted){try{control?.resume();}catch(error){runner?.stop(error.message);}}});
  details();renderQueue();renderMap();renderPermission();
  if(!demo){root.hidden=true;opener.hidden=false;message('最新状態を取得してから予定を確認してください。');}
  if(control){try{if(control.resume())message('前の画面の自動実行を引き継ぎました。');}catch(error){runner.stop('実行状態の復元失敗');message(error.message);}}
})();
