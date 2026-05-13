<!DOCTYPE html>
<html lang="zh-TW">
<head>
  <meta charset="UTF-8"/>
  <meta name="viewport" content="width=device-width,initial-scale=1.0,maximum-scale=1.0"/>
  <title>⚔️ RPG Life Tracker</title>
  <link rel="stylesheet" href="https://fonts.googleapis.com/css2?family=Orbitron:wght@400;700;900&family=Share+Tech+Mono&family=Rajdhani:wght@300;400;600;700&display=swap"/>
  <script src="https://cdnjs.cloudflare.com/ajax/libs/react/18.2.0/umd/react.production.min.js"></script>
  <script src="https://cdnjs.cloudflare.com/ajax/libs/react-dom/18.2.0/umd/react-dom.production.min.js"></script>
  <script src="https://cdnjs.cloudflare.com/ajax/libs/babel-standalone/7.23.2/babel.min.js"></script>
  <style>
    *{box-sizing:border-box;margin:0;padding:0}
    html,body,#root{height:100%;width:100%}
    body{background:#0a0a0f;color:#e0e0e0;font-family:'Rajdhani',sans-serif;overflow:hidden}
    ::-webkit-scrollbar{width:4px}
    ::-webkit-scrollbar-track{background:#111}
    ::-webkit-scrollbar-thumb{background:#00f5ff44;border-radius:2px}
    @keyframes floatXP{0%{opacity:1;transform:translateY(0) scale(1)}100%{opacity:0;transform:translateY(-80px) scale(1.4)}}
    @keyframes levelUp{0%{opacity:0;transform:scale(.5)}20%{opacity:1;transform:scale(1.2)}80%{opacity:1;transform:scale(1)}100%{opacity:0;transform:scale(1.1)}}
    @keyframes progressFlow{0%{background-position:0 0}100%{background-position:200px 0}}
    @keyframes fadeIn{from{opacity:0;transform:translateY(12px)}to{opacity:1;transform:translateY(0)}}
    @keyframes glitch{0%,100%{text-shadow:2px 0 #00f5ff,-2px 0 #ff0080}25%{text-shadow:-2px 0 #00f5ff,2px 0 #ff0080}75%{text-shadow:2px 0 #bf00ff,-2px 0 #00ff88}}
    @keyframes heartbeat{0%,100%{transform:scale(1)}50%{transform:scale(1.06)}}
    .float-xp{position:fixed;pointer-events:none;font-family:'Orbitron',sans-serif;font-weight:900;color:#00ff88;font-size:22px;animation:floatXP .9s ease-out forwards;z-index:9999;text-shadow:0 0 10px #00ff88}
    .level-up-overlay{position:fixed;inset:0;pointer-events:none;z-index:9998;display:flex;align-items:center;justify-content:center;background:radial-gradient(ellipse at center,#00f5ff11 0%,transparent 70%);animation:levelUp 2.5s ease-out forwards}
    .level-up-text{font-family:'Orbitron',sans-serif;font-size:clamp(32px,8vw,72px);font-weight:900;color:#00f5ff;text-shadow:0 0 20px #00f5ff,0 0 40px #00f5ff,0 0 80px #00f5ff44;letter-spacing:8px}
    .scan-bg{background-image:repeating-linear-gradient(0deg,transparent,transparent 2px,rgba(0,245,255,.015) 2px,rgba(0,245,255,.015) 4px)}
    input,textarea,select{outline:none}
    input[type=checkbox]{accent-color:#00f5ff;width:18px;height:18px;cursor:pointer}
  </style>
</head>
<body>
<div id="root"></div>
<script type="text/babel">
const { useState, useEffect, useCallback, useRef } = React;

const TALENTS = [
  { id:"work",    name:"工作能力", icon:"💼", color:"#00f5ff" },
  { id:"lang",    name:"語言能力", icon:"🗣️", color:"#00ff88" },
  { id:"finance", name:"財務金融", icon:"💰", color:"#ffd700" },
  { id:"body",    name:"身體健康", icon:"💪", color:"#ff4757" },
  { id:"mind",    name:"心智修練", icon:"🧠", color:"#bf00ff" },
  { id:"hobby",   name:"興趣技能", icon:"🎨", color:"#ff0080" },
];
const TALENT_TITLES = [
  {minLv:1,title:"新手學徒"},{minLv:5,title:"初階修行者"},
  {minLv:10,title:"進階掌握者"},{minLv:20,title:"精英專家"},{minLv:35,title:"傳說宗師"},
];
const CHAR_TITLES = [
  {minLv:1,title:"新手冒險者"},{minLv:6,title:"覺醒者"},
  {minLv:16,title:"精英戰士"},{minLv:31,title:"傳說英雄"},{minLv:51,title:"神話存在"},
];
const DEFAULT_HABITS = [
  {id:"h1",name:"擦防曬乳",icon:"☀️",talentId:"body",xp:5,questId:null},
  {id:"h2",name:"飯後刷牙",icon:"🦷",talentId:"body",xp:5,questId:null},
  {id:"h3",name:"睡前停用手機",icon:"📵",talentId:"mind",xp:10,questId:null},
  {id:"h4",name:"12:30前入睡",icon:"🌙",talentId:"body",xp:15,questId:null},
  {id:"h5",name:"本週戒尻",icon:"🔒",talentId:"mind",xp:20,questId:null},
];

const xpForLevel = lv => Math.floor(100*Math.pow(lv,1.5));
const calcLevel = totalXp => {
  let lv=1,acc=0;
  while(true){const n=xpForLevel(lv);if(acc+n>totalXp)break;acc+=n;lv++;}
  return {level:lv,currentXp:totalXp-acc,neededXp:xpForLevel(lv)};
};
const getTitle = (lv,table) => [...table].reverse().find(t=>lv>=t.minLv)?.title||table[0].title;
const todayStr = () => new Date().toISOString().slice(0,10);
const load = (key,def) => {try{const v=localStorage.getItem(key);return v?JSON.parse(v):def;}catch{return def;}};
const save = (key,val) => {try{localStorage.setItem(key,JSON.stringify(val));}catch{}};

const C = {
  bg:"#0a0a0f",card:"rgba(14,14,24,.85)",border:"rgba(0,245,255,.18)",
  neon:"#00f5ff",green:"#00ff88",purple:"#bf00ff",pink:"#ff0080",
  gold:"#ffd700",red:"#ff4757",text:"#e0e0e0",dim:"#666",
};

const spawnXpFloat = (xp,x,y) => {
  const el=document.createElement("div");
  el.className="float-xp";el.textContent=`+${xp} XP`;
  el.style.left=x+"px";el.style.top=y+"px";
  document.body.appendChild(el);setTimeout(()=>el.remove(),950);
};

// ── UI Primitives ──
const GlassCard = ({children,style,onClick,neonColor=C.neon}) => (
  <div onClick={onClick} style={{
    background:C.card,border:`1px solid ${neonColor}33`,borderRadius:10,
    backdropFilter:"blur(12px)",
    boxShadow:`0 0 16px ${neonColor}11,inset 0 1px 0 ${neonColor}22`,
    transition:"all .25s",cursor:onClick?"pointer":"default",...style,
  }}
    onMouseEnter={e=>{if(onClick)e.currentTarget.style.boxShadow=`0 0 28px ${neonColor}44,inset 0 1px 0 ${neonColor}44`;}}
    onMouseLeave={e=>{if(onClick)e.currentTarget.style.boxShadow=`0 0 16px ${neonColor}11,inset 0 1px 0 ${neonColor}22`;}}
  >{children}</div>
);

const NeonBtn = ({children,onClick,color=C.neon,style,small}) => (
  <button onClick={onClick} style={{
    background:"transparent",border:`1px solid ${color}`,color,
    fontFamily:"'Orbitron',sans-serif",fontWeight:700,
    fontSize:small?11:13,letterSpacing:1,
    padding:small?"5px 12px":"9px 20px",borderRadius:6,cursor:"pointer",
    boxShadow:`0 0 8px ${color}55`,transition:"all .2s",textTransform:"uppercase",...style,
  }}
    onMouseEnter={e=>{e.currentTarget.style.background=color+"22";e.currentTarget.style.boxShadow=`0 0 18px ${color}`;}}
    onMouseLeave={e=>{e.currentTarget.style.background="transparent";e.currentTarget.style.boxShadow=`0 0 8px ${color}55`;}}
  >{children}</button>
);

const XpBar = ({current,needed,color,height=8}) => {
  const pct=Math.min(100,(current/(needed||1))*100);
  return (
    <div style={{background:"#111",borderRadius:99,overflow:"hidden",height}}>
      <div style={{width:`${pct}%`,height:"100%",borderRadius:99,
        background:`linear-gradient(90deg,${color}88,${color})`,
        backgroundSize:"200px 100%",animation:"progressFlow 2s linear infinite",
        boxShadow:`0 0 8px ${color}`,transition:"width .5s ease"}}/>
    </div>
  );
};

const Badge = ({label,color}) => (
  <span style={{background:color+"22",border:`1px solid ${color}66`,color,
    borderRadius:99,padding:"2px 8px",fontSize:10,
    fontFamily:"'Share Tech Mono',monospace",whiteSpace:"nowrap"}}>{label}</span>
);

const Inp = ({value,onChange,placeholder,style,multiline,rows=3}) => {
  const s={background:"#0d0d1a",border:`1px solid ${C.border}`,color:C.text,
    fontFamily:"'Rajdhani',sans-serif",fontSize:15,
    padding:"8px 12px",borderRadius:6,width:"100%",transition:"border-color .2s",...style};
  return multiline
    ? <textarea value={value} onChange={onChange} placeholder={placeholder} rows={rows}
        style={{...s,resize:"vertical"}}
        onFocus={e=>e.target.style.borderColor=C.neon}
        onBlur={e=>e.target.style.borderColor=C.border}/>
    : <input value={value} onChange={onChange} placeholder={placeholder} style={s}
        onFocus={e=>e.target.style.borderColor=C.neon}
        onBlur={e=>e.target.style.borderColor=C.border}/>;
};

const Sel = ({value,onChange,children,style}) => (
  <select value={value} onChange={onChange} style={{
    background:"#0d0d1a",border:`1px solid ${C.border}`,color:C.text,
    fontFamily:"'Rajdhani',sans-serif",fontSize:15,
    padding:"8px 12px",borderRadius:6,width:"100%",cursor:"pointer",...style}}>{children}</select>
);

const Modal = ({title,onClose,children,width=480}) => (
  <div style={{position:"fixed",inset:0,background:"#000a",zIndex:1000,display:"flex",alignItems:"center",justifyContent:"center",padding:16}}
    onClick={e=>e.target===e.currentTarget&&onClose()}>
    <GlassCard style={{width:"100%",maxWidth:width,maxHeight:"90vh",overflow:"auto",padding:24,animation:"fadeIn .25s ease"}}>
      <div style={{display:"flex",justifyContent:"space-between",alignItems:"center",marginBottom:20}}>
        <div style={{fontFamily:"'Orbitron',sans-serif",fontSize:16,color:C.neon,textShadow:`0 0 10px ${C.neon}`}}>{title}</div>
        <button onClick={onClose} style={{background:"none",border:"none",color:C.dim,fontSize:22,cursor:"pointer"}}>✕</button>
      </div>
      {children}
    </GlassCard>
  </div>
);

const LevelUpOverlay = ({level,onDone}) => {
  useEffect(()=>{const t=setTimeout(onDone,2600);return()=>clearTimeout(t);},[]);
  return (
    <div className="level-up-overlay" style={{flexDirection:"column",gap:16}}>
      <div className="level-up-text" style={{animation:"glitch 0.3s infinite"}}>LEVEL UP!</div>
      <div style={{fontFamily:"'Orbitron',sans-serif",fontSize:28,color:C.gold,textShadow:`0 0 20px ${C.gold}`}}>LEVEL {level}</div>
    </div>
  );
};

// ══════════════════════════════════════════════
// APP
// ══════════════════════════════════════════════
function App() {
  const [tab,setTab] = useState(0);
  const [char,setChar] = useState(()=>load("rpg_character",{name:"玩家一號",motto:"每天進步一點，終將成為傳說。"}));
  const [talents,setTalents] = useState(()=>load("rpg_talents",TALENTS.map(t=>({...t,xp:0}))));
  const [quests,setQuests] = useState(()=>load("rpg_quests",[]));
  const [habits,setHabits] = useState(()=>load("rpg_habits",DEFAULT_HABITS));
  const [dailyRec,setDailyRec] = useState(()=>load(`rpg_daily_${todayStr()}`,{}));
  const [note,setNote] = useState(()=>load(`rpg_notes_${todayStr()}`,""));
  const [tempTasks,setTempTasks] = useState(()=>load(`rpg_temp_${todayStr()}`,[]))
  const [levelUpShow,setLevelUpShow] = useState(null);
  const prevLvRef = useRef(null);

  useEffect(()=>save("rpg_character",char),[char]);
  useEffect(()=>save("rpg_talents",talents),[talents]);
  useEffect(()=>save("rpg_quests",quests),[quests]);
  useEffect(()=>save("rpg_habits",habits),[habits]);
  useEffect(()=>save(`rpg_daily_${todayStr()}`,dailyRec),[dailyRec]);
  useEffect(()=>save(`rpg_notes_${todayStr()}`,note),[note]);
  useEffect(()=>save(`rpg_temp_${todayStr()}`,tempTasks),[tempTasks]);

  const totalXp = talents.reduce((s,t)=>s+(t.xp||0),0);
  const {level,currentXp,neededXp} = calcLevel(totalXp);

  useEffect(()=>{
    if(prevLvRef.current!==null&&level>prevLvRef.current)setLevelUpShow(level);
    prevLvRef.current=level;
  },[level]);

  const calcStreak = () => {
    let s=0,d=new Date();
    while(true){
      const key=d.toISOString().slice(0,10);
      const rec=load(`rpg_daily_${key}`,{});
      const ah=load("rpg_habits",[]);
      if(!ah.length)break;
      if(ah.filter(h=>rec[h.id]).length<ah.length)break;
      s++;d.setDate(d.getDate()-1);
    }
    return s;
  };
  const streak = calcStreak();

  const completeTask = useCallback((id,xp,talentId,questId,e)=>{
    if(dailyRec[id]){
      setDailyRec(r=>{const n={...r};delete n[id];return n;});
      setTalents(ts=>ts.map(t=>t.id===talentId?{...t,xp:Math.max(0,(t.xp||0)-xp)}:t));
    } else {
      const rect=e?.currentTarget?.getBoundingClientRect();
      if(rect)spawnXpFloat(xp,rect.left+rect.width/2,rect.top);
      setDailyRec(r=>({...r,[id]:true}));
      setTalents(ts=>ts.map(t=>t.id===talentId?{...t,xp:(t.xp||0)+xp}:t));
      // ─ Quest score progress ─
      if(questId){
        setQuests(qs=>qs.map(q=>{
          if(q.id!==questId)return q;
          const newScore=(q.currentScore||0)+xp;
          const isDone=newScore>=(q.targetScore||100);
          return {...q,currentScore:newScore,status:isDone?"done":q.status};
        }));
      }
    }
  },[dailyRec]);

  const TABS=[
    {icon:"⚔️",label:"角色"},{icon:"📋",label:"今日"},
    {icon:"📅",label:"日曆"},{icon:"🎯",label:"目標"},{icon:"⚙️",label:"設定"},
  ];

  return (
    <div className="scan-bg" style={{height:"100vh",display:"flex",flexDirection:"column",background:C.bg,position:"relative",overflow:"hidden"}}>
      <div style={{position:"absolute",top:-100,left:-100,width:400,height:400,background:"radial-gradient(circle,#00f5ff08,transparent 70%)",pointerEvents:"none"}}/>
      <div style={{position:"absolute",bottom:-80,right:-80,width:300,height:300,background:"radial-gradient(circle,#bf00ff08,transparent 70%)",pointerEvents:"none"}}/>
      <div style={{flex:1,overflow:"auto",position:"relative"}}>
        {tab===0&&<CharPage char={char} setChar={setChar} talents={talents} level={level} currentXp={currentXp} neededXp={neededXp} totalXp={totalXp} streak={streak} quests={quests} setQuests={setQuests}/>}
        {tab===1&&<TodayPage habits={habits} setHabits={setHabits} tempTasks={tempTasks} setTempTasks={setTempTasks} dailyRec={dailyRec} note={note} setNote={setNote} completeTask={completeTask} quests={quests} setQuests={setQuests}/>}
        {tab===2&&<CalendarPage habits={habits}/>}
        {tab===3&&<QuestPage quests={quests} setQuests={setQuests} talents={talents} habits={habits} setHabits={setHabits}/>}
        {tab===4&&<SettingsPage char={char} setChar={setChar} talents={talents} setTalents={setTalents} setHabits={setHabits} setQuests={setQuests} setDailyRec={setDailyRec}/>}
      </div>
      <div style={{display:"flex",background:"rgba(8,8,16,.96)",borderTop:`1px solid ${C.neon}22`,backdropFilter:"blur(20px)"}}>
        {TABS.map((t,i)=>(
          <button key={i} onClick={()=>setTab(i)} style={{
            flex:1,background:"none",border:"none",cursor:"pointer",
            padding:"10px 4px 8px",display:"flex",flexDirection:"column",alignItems:"center",gap:3,
            color:tab===i?C.neon:C.dim,transition:"color .2s",
            borderTop:tab===i?`2px solid ${C.neon}`:"2px solid transparent",
          }}>
            <span style={{fontSize:20}}>{t.icon}</span>
            <span style={{fontFamily:"'Share Tech Mono',monospace",fontSize:10,letterSpacing:1}}>{t.label}</span>
          </button>
        ))}
      </div>
      {levelUpShow&&<LevelUpOverlay level={levelUpShow} onDone={()=>setLevelUpShow(null)}/>}
    </div>
  );
}

// ══════════════════════════════════════════════
// PAGE 1 — CHARACTER
// ══════════════════════════════════════════════
function CharPage({char,setChar,talents,level,currentXp,neededXp,totalXp,streak,quests,setQuests}) {
  const [editName,setEditName]=useState(false);
  const [editMotto,setEditMotto]=useState(false);
  const [tmpName,setTmpName]=useState(char.name);
  const [tmpMotto,setTmpMotto]=useState(char.motto);
  const [activeTalent,setActiveTalent]=useState(null);
  const charTitle=getTitle(level,CHAR_TITLES);

  const saveField=(field,val)=>{
    const updated={...char,[field]:val};
    save("rpg_character",updated);
    setChar(updated);
    if(field==="name")setEditName(false);
    if(field==="motto")setEditMotto(false);
  };

  return (
    <div style={{padding:"16px 16px 8px"}}>
      <GlassCard neonColor={C.neon} style={{padding:20,marginBottom:16,position:"relative",overflow:"hidden"}}>
        <div style={{position:"absolute",top:0,right:0,width:120,height:120,opacity:.06,
          backgroundImage:`url("data:image/svg+xml,%3Csvg xmlns='http://www.w3.org/2000/svg' width='120' height='120'%3E%3Crect x='10' y='10' width='100' height='100' fill='none' stroke='%2300f5ff' stroke-width='1'/%3E%3Cline x1='10' y1='50' x2='110' y2='50' stroke='%2300f5ff' stroke-width='1'/%3E%3Cline x1='60' y1='10' x2='60' y2='110' stroke='%2300f5ff' stroke-width='1'/%3E%3Ccircle cx='60' cy='50' r='8' fill='none' stroke='%2300f5ff' stroke-width='1'/%3E%3C/svg%3E")`}}/>
        <div style={{display:"flex",gap:16,alignItems:"flex-start"}}>
          <div style={{flexShrink:0,width:72,height:72,border:`2px solid ${C.neon}`,borderRadius:8,display:"flex",alignItems:"center",justifyContent:"center",background:"#0d1a1a",boxShadow:`0 0 16px ${C.neon}44`,position:"relative",animation:"heartbeat 3s infinite"}}>
            <pre style={{color:C.neon,fontSize:9,lineHeight:1.3,fontFamily:"'Share Tech Mono',monospace"}}>{`╔══╗\n║>_║\n╚══╝\n ▓▓▓`}</pre>
            <div style={{position:"absolute",bottom:-2,right:-2,background:C.gold,borderRadius:99,width:20,height:20,display:"flex",alignItems:"center",justifyContent:"center",fontSize:10,fontFamily:"'Orbitron',sans-serif",fontWeight:900,color:"#000"}}>{level}</div>
          </div>
          <div style={{flex:1,minWidth:0}}>
            <div style={{display:"flex",alignItems:"center",gap:8,marginBottom:4}}>
              {editName?(
                <div style={{display:"flex",gap:8,flex:1}}>
                  <Inp value={tmpName} onChange={e=>setTmpName(e.target.value)} style={{fontSize:13}}/>
                  <NeonBtn small onClick={()=>saveField("name",tmpName)}>✓</NeonBtn>
                  <NeonBtn small onClick={()=>setEditName(false)} color={C.dim}>✕</NeonBtn>
                </div>
              ):(
                <>
                  <div style={{fontFamily:"'Orbitron',sans-serif",fontSize:18,fontWeight:900,color:"#fff",textShadow:`0 0 10px ${C.neon}66`,whiteSpace:"nowrap",overflow:"hidden",textOverflow:"ellipsis"}}>{char.name}</div>
                  <button onClick={()=>setEditName(true)} style={{background:"none",border:"none",color:C.dim,cursor:"pointer",fontSize:14}}>✏️</button>
                </>
              )}
            </div>
            <div style={{display:"flex",gap:8,marginBottom:8,flexWrap:"wrap"}}>
              <Badge label={charTitle} color={C.neon}/>
              <Badge label={`🔥 ${streak}天連線`} color={C.gold}/>
            </div>
            {editMotto?(
              <div style={{display:"flex",gap:8}}>
                <Inp value={tmpMotto} onChange={e=>setTmpMotto(e.target.value)} style={{fontSize:12}}/>
                <NeonBtn small onClick={()=>saveField("motto",tmpMotto)}>✓</NeonBtn>
                <NeonBtn small onClick={()=>setEditMotto(false)} color={C.dim}>✕</NeonBtn>
              </div>
            ):(
              <div onClick={()=>setEditMotto(true)} style={{color:C.dim,fontStyle:"italic",fontSize:13,cursor:"pointer"}}>「{char.motto}」</div>
            )}
          </div>
        </div>
        <div style={{marginTop:14}}>
          <div style={{display:"flex",justifyContent:"space-between",marginBottom:5}}>
            <span style={{fontFamily:"'Share Tech Mono',monospace",fontSize:11,color:C.neon}}>TOTAL XP</span>
            <span style={{fontFamily:"'Share Tech Mono',monospace",fontSize:11,color:C.dim}}>{currentXp} / {neededXp}</span>
          </div>
          <XpBar current={currentXp} needed={neededXp} color={C.neon} height={10}/>
          <div style={{fontFamily:"'Share Tech Mono',monospace",fontSize:10,color:C.dim,marginTop:4,textAlign:"right"}}>TOTAL: {totalXp} XP</div>
        </div>
      </GlassCard>

      <div style={{fontFamily:"'Orbitron',sans-serif",fontSize:11,color:C.dim,letterSpacing:2,marginBottom:10}}>▸ TALENT MATRIX</div>
      <div style={{display:"grid",gridTemplateColumns:"1fr 1fr",gap:10}}>
        {talents.map(t=>{
          const {level:tLv,currentXp:cxp,neededXp:nxp}=calcLevel(t.xp||0);
          const tTitle=getTitle(tLv,TALENT_TITLES);
          const tq=quests.filter(q=>q.talentId===t.id);
          return (
            <GlassCard key={t.id} neonColor={t.color} onClick={()=>setActiveTalent(t)} style={{padding:14,cursor:"pointer"}}>
              <div style={{display:"flex",justifyContent:"space-between",alignItems:"center",marginBottom:6}}>
                <span style={{fontSize:22}}>{t.icon}</span>
                <span style={{fontFamily:"'Orbitron',sans-serif",fontSize:11,fontWeight:700,color:t.color}}>Lv.{tLv}</span>
              </div>
              <div style={{fontFamily:"'Rajdhani',sans-serif",fontWeight:700,fontSize:14,color:"#fff",marginBottom:2}}>{t.name}</div>
              <div style={{color:t.color,fontSize:10,fontFamily:"'Share Tech Mono',monospace",marginBottom:6}}>{tTitle}</div>
              <XpBar current={cxp} needed={nxp} color={t.color}/>
              <div style={{fontSize:10,color:C.dim,marginTop:4,fontFamily:"'Share Tech Mono',monospace"}}>{cxp}/{nxp} · {tq.length} 主線</div>
            </GlassCard>
          );
        })}
      </div>
      {activeTalent&&<TalentModal talent={activeTalent} quests={quests} setQuests={setQuests} onClose={()=>setActiveTalent(null)}/>}
    </div>
  );
}

function TalentModal({talent,quests,setQuests,onClose}) {
  const [addOpen,setAddOpen]=useState(false);
  // [1] targetScore field added to quest form
  const [form,setForm]=useState({name:"",desc:"",deadline:"",targetScore:100});
  const tq=quests.filter(q=>q.talentId===talent.id);

  const addQuest=()=>{
    if(!form.name.trim())return;
    setQuests(qs=>[...qs,{
      id:Date.now().toString(),...form,
      talentId:talent.id,status:"active",
      currentScore:0,targetScore:Number(form.targetScore)||100,
      subs:[],
    }]);
    setForm({name:"",desc:"",deadline:"",targetScore:100});
    setAddOpen(false);
  };

  return (
    <Modal title={`${talent.icon} ${talent.name} — 主線任務`} onClose={onClose} width={500}>
      {tq.length===0&&<div style={{color:C.dim,textAlign:"center",padding:24}}>尚無主線任務</div>}
      {tq.map(q=>{
        const prog=Math.min(100,((q.currentScore||0)/(q.targetScore||100))*100);
        return (
          <GlassCard key={q.id} neonColor={talent.color} style={{padding:14,marginBottom:10}}>
            <div style={{fontWeight:700,fontSize:15,color:"#fff",marginBottom:4}}>{q.name}</div>
            {q.desc&&<div style={{color:C.dim,fontSize:13,marginBottom:6}}>{q.desc}</div>}
            <div style={{display:"flex",gap:8,flexWrap:"wrap",marginBottom:8}}>
              {q.deadline&&<Badge label={`📅 ${q.deadline}`} color={C.dim}/>}
              <Badge label={`🎯 ${q.currentScore||0}/${q.targetScore||100} 分`} color={talent.color}/>
              <Badge label={q.status==="done"?"🟢 完成":q.currentScore>0?"🟡 進行中":"🔴 未開始"} color={q.status==="done"?C.green:q.currentScore>0?C.gold:C.red}/>
            </div>
            <XpBar current={q.currentScore||0} needed={q.targetScore||100} color={talent.color}/>
            <div style={{fontSize:10,color:C.dim,marginTop:4,fontFamily:"'Share Tech Mono',monospace"}}>
              進度：{prog.toFixed(0)}%
            </div>
          </GlassCard>
        );
      })}
      {addOpen?(
        <GlassCard style={{padding:14}}>
          <div style={{display:"flex",flexDirection:"column",gap:10}}>
            <Inp value={form.name} onChange={e=>setForm(f=>({...f,name:e.target.value}))} placeholder="任務名稱（必填）"/>
            <Inp value={form.desc} onChange={e=>setForm(f=>({...f,desc:e.target.value}))} placeholder="描述 / 備註"/>
            <div style={{display:"grid",gridTemplateColumns:"1fr 1fr",gap:10}}>
              <Inp value={form.deadline} onChange={e=>setForm(f=>({...f,deadline:e.target.value}))} placeholder="截止日 YYYY-MM-DD"/>
              <div>
                <div style={{color:C.dim,fontSize:11,marginBottom:4}}>目標分數（累積達到即完成）</div>
                <Inp value={form.targetScore} onChange={e=>setForm(f=>({...f,targetScore:e.target.value}))} placeholder="100"/>
              </div>
            </div>
            <div style={{display:"flex",gap:10}}>
              <NeonBtn onClick={addQuest} color={talent.color}>新增</NeonBtn>
              <NeonBtn onClick={()=>setAddOpen(false)} color={C.dim}>取消</NeonBtn>
            </div>
          </div>
        </GlassCard>
      ):(
        <NeonBtn onClick={()=>setAddOpen(true)} color={talent.color} style={{width:"100%",marginTop:8}}>+ 新增主線任務</NeonBtn>
      )}
    </Modal>
  );
}

// ══════════════════════════════════════════════
// PAGE 2 — TODAY
// ══════════════════════════════════════════════
function TodayPage({habits,setHabits,tempTasks,setTempTasks,dailyRec,note,setNote,completeTask,quests,setQuests}) {
  const [addHabit,setAddHabit]=useState(false);
  const [addTemp,setAddTemp]=useState(false);
  // [2] temp task can now pick a quest habit (one not already in habits)
  const [hForm,setHForm]=useState({name:"",icon:"⭐",talentId:"body",xp:10,questId:""});
  const [tForm,setTForm]=useState({mode:"new",name:"",icon:"⭐",talentId:"body",xp:20,questId:"",selectedQuestHabit:""});

  const allTasks=[...habits,...tempTasks];
  const todayXp=allTasks.reduce((s,t)=>dailyRec[t.id]?s+(t.xp||0):s,0);
  const doneTasks=allTasks.filter(t=>dailyRec[t.id]).length;
  const totalTasks=allTasks.length;
  const dateStr=new Date().toLocaleDateString("zh-TW",{year:"numeric",month:"long",day:"numeric",weekday:"long"});

  // Quests that have linked habits not already in the habits list
  const linkedHabitsInHabits = habits.map(h=>h.questHabitId).filter(Boolean);
  // Quests with a questHabit defined
  const questsWithHabits = quests.filter(q=>q.questHabit && !linkedHabitsInHabits.includes(q.id+"_habit"));

  const addHabitFn=()=>{
    if(!hForm.name.trim())return;
    setHabits(hs=>[...hs,{...hForm,id:Date.now().toString()}]);
    setHForm({name:"",icon:"⭐",talentId:"body",xp:10,questId:""});
    setAddHabit(false);
  };

  const addTempFn=()=>{
    if(tForm.mode==="quest"){
      // Find the quest and add a temp task linked to it
      const q=quests.find(x=>x.id===tForm.selectedQuestHabit);
      if(!q)return;
      const t=TALENTS.find(x=>x.id===q.talentId)||TALENTS[0];
      setTempTasks(ts=>[...ts,{
        id:Date.now().toString(),
        name:q.name,
        icon:"🎯",
        talentId:q.talentId,
        xp:Math.min(50,Math.floor((q.targetScore||100)/5)),
        questId:q.id,
      }]);
    } else {
      if(!tForm.name.trim())return;
      setTempTasks(ts=>[...ts,{
        id:Date.now().toString(),
        name:tForm.name,icon:tForm.icon||"📌",
        talentId:tForm.talentId,xp:tForm.xp,
        questId:tForm.questId||null,
      }]);
    }
    setTForm({mode:"new",name:"",icon:"⭐",talentId:"body",xp:20,questId:"",selectedQuestHabit:""});
    setAddTemp(false);
  };

  return (
    <div style={{padding:16}}>
      {/* Overview card */}
      <GlassCard style={{padding:16,marginBottom:14}}>
        <div style={{fontFamily:"'Share Tech Mono',monospace",fontSize:11,color:C.dim,marginBottom:8}}>{dateStr}</div>
        <div style={{display:"flex",justifyContent:"space-between",alignItems:"center",marginBottom:10}}>
          <div>
            <div style={{fontFamily:"'Orbitron',sans-serif",fontSize:24,fontWeight:900,color:C.green,textShadow:`0 0 12px ${C.green}`}}>+{todayXp} XP</div>
            <div style={{color:C.dim,fontSize:13}}>今日獲得</div>
          </div>
          <div style={{textAlign:"right"}}>
            <div style={{fontFamily:"'Orbitron',sans-serif",fontSize:22,color:C.neon}}>{doneTasks}<span style={{color:C.dim,fontSize:14}}>/{totalTasks}</span></div>
            <div style={{color:C.dim,fontSize:13}}>任務完成</div>
          </div>
        </div>
        <XpBar current={doneTasks} needed={totalTasks||1} color={C.neon} height={10}/>
        <div style={{textAlign:"right",fontSize:10,color:C.dim,marginTop:4,fontFamily:"'Share Tech Mono',monospace"}}>
          {totalTasks?((doneTasks/totalTasks)*100).toFixed(0):0}% COMPLETE
        </div>
      </GlassCard>

      {/* Fixed Habits */}
      <div style={{display:"flex",justifyContent:"space-between",alignItems:"center",marginBottom:8}}>
        <div style={{fontFamily:"'Orbitron',sans-serif",fontSize:11,color:C.dim,letterSpacing:2}}>▸ 固定習慣</div>
        <NeonBtn small onClick={()=>setAddHabit(!addHabit)}>+ 新增</NeonBtn>
      </div>
      {addHabit&&(
        <GlassCard style={{padding:14,marginBottom:10}}>
          <div style={{display:"flex",flexDirection:"column",gap:8}}>
            <div style={{display:"grid",gridTemplateColumns:"60px 1fr",gap:8}}>
              <Inp value={hForm.icon} onChange={e=>setHForm(f=>({...f,icon:e.target.value}))} placeholder="Icon"/>
              <Inp value={hForm.name} onChange={e=>setHForm(f=>({...f,name:e.target.value}))} placeholder="習慣名稱"/>
            </div>
            <div style={{display:"grid",gridTemplateColumns:"1fr 1fr",gap:8}}>
              <Sel value={hForm.talentId} onChange={e=>setHForm(f=>({...f,talentId:e.target.value}))}>
                {TALENTS.map(t=><option key={t.id} value={t.id}>{t.icon} {t.name}</option>)}
              </Sel>
              <Inp value={hForm.xp} onChange={e=>setHForm(f=>({...f,xp:+e.target.value}))} placeholder="XP值"/>
            </div>
            {/* Link to quest */}
            <div>
              <div style={{color:C.dim,fontSize:11,marginBottom:4}}>連結主線任務（可選）</div>
              <Sel value={hForm.questId} onChange={e=>setHForm(f=>({...f,questId:e.target.value}))}>
                <option value="">— 無 —</option>
                {quests.filter(q=>q.status!=="done").map(q=>{
                  const t=TALENTS.find(x=>x.id===q.talentId)||TALENTS[0];
                  return <option key={q.id} value={q.id}>{t.icon} {q.name}</option>;
                })}
              </Sel>
            </div>
            <div style={{display:"flex",gap:8}}>
              <NeonBtn color={C.green} onClick={addHabitFn}>新增習慣</NeonBtn>
              <NeonBtn color={C.dim} onClick={()=>setAddHabit(false)}>取消</NeonBtn>
            </div>
          </div>
        </GlassCard>
      )}
      {habits.map(h=><HabitRow key={h.id} item={h} done={!!dailyRec[h.id]} onToggle={completeTask} onDelete={()=>setHabits(hs=>hs.filter(x=>x.id!==h.id))}/>)}

      {/* [2] Temp Tasks — can add new or pick from quests */}
      <div style={{display:"flex",justifyContent:"space-between",alignItems:"center",margin:"14px 0 8px"}}>
        <div style={{fontFamily:"'Orbitron',sans-serif",fontSize:11,color:C.dim,letterSpacing:2}}>▸ 今日 TO DO</div>
        <NeonBtn small color={C.purple} onClick={()=>setAddTemp(!addTemp)}>+ 新增</NeonBtn>
      </div>
      {addTemp&&(
        <GlassCard neonColor={C.purple} style={{padding:14,marginBottom:10}}>
          {/* Mode toggle */}
          <div style={{display:"flex",gap:8,marginBottom:12}}>
            <NeonBtn small color={tForm.mode==="new"?C.purple:C.dim} onClick={()=>setTForm(f=>({...f,mode:"new"}))}>✏️ 自訂任務</NeonBtn>
            <NeonBtn small color={tForm.mode==="quest"?C.gold:C.dim} onClick={()=>setTForm(f=>({...f,mode:"quest"}))}>🎯 從目標選</NeonBtn>
          </div>
          {tForm.mode==="new"?(
            <div style={{display:"flex",flexDirection:"column",gap:8}}>
              <div style={{display:"grid",gridTemplateColumns:"60px 1fr",gap:8}}>
                <Inp value={tForm.icon} onChange={e=>setTForm(f=>({...f,icon:e.target.value}))} placeholder="Icon"/>
                <Inp value={tForm.name} onChange={e=>setTForm(f=>({...f,name:e.target.value}))} placeholder="任務名稱"/>
              </div>
              <div style={{display:"grid",gridTemplateColumns:"1fr 1fr",gap:8}}>
                <Sel value={tForm.talentId} onChange={e=>setTForm(f=>({...f,talentId:e.target.value}))}>
                  {TALENTS.map(t=><option key={t.id} value={t.id}>{t.icon} {t.name}</option>)}
                </Sel>
                <Inp value={tForm.xp} onChange={e=>setTForm(f=>({...f,xp:+e.target.value}))} placeholder="XP值"/>
              </div>
              <div>
                <div style={{color:C.dim,fontSize:11,marginBottom:4}}>連結主線任務（可選）</div>
                <Sel value={tForm.questId} onChange={e=>setTForm(f=>({...f,questId:e.target.value}))}>
                  <option value="">— 無 —</option>
                  {quests.filter(q=>q.status!=="done").map(q=>{
                    const t=TALENTS.find(x=>x.id===q.talentId)||TALENTS[0];
                    return <option key={q.id} value={q.id}>{t.icon} {q.name}</option>;
                  })}
                </Sel>
              </div>
            </div>
          ):(
            <div style={{display:"flex",flexDirection:"column",gap:8}}>
              <div style={{color:C.dim,fontSize:12,marginBottom:4}}>選擇進行中的主線任務</div>
              {quests.filter(q=>q.status!=="done").length===0
                ?<div style={{color:C.dim,fontSize:13,textAlign:"center",padding:"8px 0"}}>目前沒有進行中的目標</div>
                :quests.filter(q=>q.status!=="done").map(q=>{
                  const t=TALENTS.find(x=>x.id===q.talentId)||TALENTS[0];
                  const prog=Math.min(100,((q.currentScore||0)/(q.targetScore||100))*100);
                  const selected=tForm.selectedQuestHabit===q.id;
                  return (
                    <GlassCard key={q.id} neonColor={selected?t.color:C.dim}
                      onClick={()=>setTForm(f=>({...f,selectedQuestHabit:f.selectedQuestHabit===q.id?"":q.id}))}
                      style={{padding:"10px 14px",border:`1px solid ${selected?t.color+"66":C.border}`}}>
                      <div style={{display:"flex",justifyContent:"space-between",alignItems:"center"}}>
                        <div>
                          <div style={{fontSize:14,fontWeight:600,color:"#fff"}}>{t.icon} {q.name}</div>
                          <div style={{fontSize:10,color:C.dim,fontFamily:"'Share Tech Mono',monospace",marginTop:3}}>
                            {q.currentScore||0}/{q.targetScore||100} 分 · {prog.toFixed(0)}%
                          </div>
                        </div>
                        {selected&&<span style={{color:t.color,fontSize:18}}>✓</span>}
                      </div>
                      <div style={{marginTop:6}}><XpBar current={q.currentScore||0} needed={q.targetScore||100} color={t.color} height={4}/></div>
                    </GlassCard>
                  );
                })
              }
            </div>
          )}
          <div style={{display:"flex",gap:8,marginTop:12}}>
            <NeonBtn color={C.purple} onClick={addTempFn}
              style={{opacity:(tForm.mode==="new"&&!tForm.name.trim())||(tForm.mode==="quest"&&!tForm.selectedQuestHabit)?.5:1}}>
              新增
            </NeonBtn>
            <NeonBtn color={C.dim} onClick={()=>setAddTemp(false)}>取消</NeonBtn>
          </div>
        </GlassCard>
      )}
      {tempTasks.length===0&&!addTemp&&(
        <div style={{color:C.dim,fontSize:13,textAlign:"center",padding:"12px 0"}}>點擊「+ 新增」加入今日 TO DO</div>
      )}
      {tempTasks.map(t=><HabitRow key={t.id} item={t} done={!!dailyRec[t.id]} onToggle={completeTask} onDelete={()=>setTempTasks(ts=>ts.filter(x=>x.id!==t.id))} questName={quests.find(q=>q.id===t.questId)?.name}/>)}

      {/* Daily Note */}
      <div style={{fontFamily:"'Orbitron',sans-serif",fontSize:11,color:C.dim,letterSpacing:2,margin:"14px 0 8px"}}>▸ 今日反省</div>
      <Inp multiline value={note} onChange={e=>setNote(e.target.value)} placeholder="記錄今日心得、反省、學習..." rows={4}/>
    </div>
  );
}

function HabitRow({item,done,onToggle,onDelete,questName}) {
  const talent=TALENTS.find(t=>t.id===item.talentId)||TALENTS[0];
  return (
    <GlassCard neonColor={talent.color} style={{padding:"10px 14px",marginBottom:8,display:"flex",alignItems:"center",gap:12,opacity:done?.7:1,transition:"opacity .3s"}}>
      <input type="checkbox" checked={done} onChange={e=>onToggle(item.id,item.xp,item.talentId,item.questId,e)}/>
      <span style={{fontSize:18}}>{item.icon||"📌"}</span>
      <div style={{flex:1,minWidth:0}}>
        <div style={{fontWeight:600,fontSize:14,color:done?C.dim:"#fff",textDecoration:done?"line-through":"none",transition:"color .3s"}}>{item.name}</div>
        <div style={{display:"flex",gap:6,flexWrap:"wrap",marginTop:2}}>
          <Badge label={talent.name} color={talent.color}/>
          {questName&&<Badge label={`🎯 ${questName}`} color={C.gold}/>}
        </div>
      </div>
      <div style={{fontFamily:"'Share Tech Mono',monospace",fontSize:12,color:talent.color,whiteSpace:"nowrap"}}>+{item.xp} XP</div>
      <button onClick={onDelete} style={{background:"none",border:"none",color:C.dim,cursor:"pointer",fontSize:16,padding:"0 4px"}}>✕</button>
    </GlassCard>
  );
}

// ══════════════════════════════════════════════
// PAGE 3 — CALENDAR
// ══════════════════════════════════════════════
function CalendarPage({habits}) {
  const [viewDate,setViewDate]=useState(new Date());
  const [selectedDay,setSelectedDay]=useState(null);
  const yr=viewDate.getFullYear(),mo=viewDate.getMonth();
  const firstDay=new Date(yr,mo,1).getDay();
  const daysInMonth=new Date(yr,mo+1,0).getDate();
  const WD=["日","一","二","三","四","五","六"];

  const getDayData=d=>{
    const key=`${yr}-${String(mo+1).padStart(2,"0")}-${String(d).padStart(2,"0")}`;
    const rec=load(`rpg_daily_${key}`,{});
    const ah=load("rpg_habits",[]);
    const done=ah.filter(h=>rec[h.id]).length;
    const xp=ah.reduce((s,h)=>rec[h.id]?s+(h.xp||0):s,0);
    return {pct:ah.length?done/ah.length:0,xp};
  };

  return (
    <div style={{padding:16}}>
      <div style={{display:"flex",justifyContent:"space-between",alignItems:"center",marginBottom:16}}>
        <NeonBtn small onClick={()=>setViewDate(new Date(yr,mo-1))}>‹</NeonBtn>
        <div style={{fontFamily:"'Orbitron',sans-serif",fontSize:18,color:C.neon,textShadow:`0 0 10px ${C.neon}`}}>{yr}年 {mo+1}月</div>
        <NeonBtn small onClick={()=>setViewDate(new Date(yr,mo+1))}>›</NeonBtn>
      </div>
      <div style={{display:"grid",gridTemplateColumns:"repeat(7,1fr)",gap:2,marginBottom:4}}>
        {WD.map(w=><div key={w} style={{textAlign:"center",fontSize:11,color:C.dim,fontFamily:"'Share Tech Mono',monospace",padding:"4px 0"}}>{w}</div>)}
      </div>
      <div style={{display:"grid",gridTemplateColumns:"repeat(7,1fr)",gap:3}}>
        {Array(firstDay).fill(null).map((_,i)=><div key={`e${i}`}/>)}
        {Array(daysInMonth).fill(null).map((_,i)=>{
          const d=i+1,{pct,xp}=getDayData(d);
          const isToday=new Date().getDate()===d&&new Date().getMonth()===mo&&new Date().getFullYear()===yr;
          const alpha=Math.floor(pct*180).toString(16).padStart(2,"0");
          // [3] Check if has diary note
          const noteKey=`${yr}-${String(mo+1).padStart(2,"0")}-${String(d).padStart(2,"0")}`;
          const hasNote=!!load(`rpg_notes_${noteKey}`,"");
          return (
            <div key={d} onClick={()=>setSelectedDay(d)} style={{
              aspectRatio:"1",borderRadius:6,background:pct>0?`#00ff88${alpha}`:"transparent",
              border:isToday?`1px solid ${C.neon}`:`1px solid ${C.border}`,
              display:"flex",flexDirection:"column",alignItems:"center",justifyContent:"center",
              cursor:"pointer",transition:"all .2s",boxShadow:isToday?`0 0 8px ${C.neon}44`:"none",position:"relative",
            }}
              onMouseEnter={e=>e.currentTarget.style.borderColor=C.neon}
              onMouseLeave={e=>e.currentTarget.style.borderColor=isToday?C.neon:C.border}
            >
              <div style={{fontSize:13,color:isToday?C.neon:"#fff",fontWeight:isToday?700:400}}>{d}</div>
              {xp>0&&<div style={{fontSize:8,color:C.green,fontFamily:"'Share Tech Mono',monospace"}}>{xp}</div>}
              {hasNote&&<div style={{position:"absolute",top:2,right:3,fontSize:7,color:C.purple}}>✍</div>}
            </div>
          );
        })}
      </div>
      <div style={{display:"flex",gap:12,alignItems:"center",marginTop:12}}>
        <span style={{fontSize:11,color:C.dim}}>完成率：</span>
        {[0,.25,.5,.75,1].map(p=>{
          const a=Math.floor(p*180).toString(16).padStart(2,"0");
          return <div key={p} style={{width:16,height:16,borderRadius:3,background:p>0?`#00ff88${a}`:"#1a1a2a",border:"1px solid #333"}}/>;
        })}
        <span style={{fontSize:11,color:C.dim}}>→ 100%</span>
        <span style={{fontSize:10,color:C.purple,marginLeft:8}}>✍ 有日記</span>
      </div>
      {selectedDay&&<DayModal day={selectedDay} year={yr} month={mo} habits={habits} onClose={()=>setSelectedDay(null)}/>}
    </div>
  );
}

// [3] DayModal — note is now editable
function DayModal({day,year,month,habits,onClose}) {
  const key=`${year}-${String(month+1).padStart(2,"0")}-${String(day).padStart(2,"0")}`;
  const rec=load(`rpg_daily_${key}`,{});
  const xp=habits.reduce((s,h)=>rec[h.id]?s+(h.xp||0):s,0);
  const [noteVal,setNoteVal]=useState(()=>load(`rpg_notes_${key}`,""));
  const [saved,setSaved]=useState(false);

  const saveNote=()=>{
    save(`rpg_notes_${key}`,noteVal);
    setSaved(true);
    setTimeout(()=>setSaved(false),1500);
  };

  const isToday=key===todayStr();

  return (
    <Modal title={`${year}/${month+1}/${day} 記錄`} onClose={onClose}>
      <div style={{marginBottom:12}}><Badge label={`+${xp} XP`} color={C.green}/></div>
      <div style={{fontFamily:"'Orbitron',sans-serif",fontSize:11,color:C.dim,marginBottom:8}}>任務完成情況</div>
      {habits.map(h=>{
        const t=TALENTS.find(x=>x.id===h.talentId)||TALENTS[0];
        return (
          <div key={h.id} style={{display:"flex",alignItems:"center",gap:10,padding:"6px 0",borderBottom:`1px solid ${C.border}`}}>
            <span style={{color:rec[h.id]?C.green:C.red,fontSize:16}}>{rec[h.id]?"✓":"✗"}</span>
            <span style={{fontSize:14}}>{h.icon} {h.name}</span>
            <Badge label={t.name} color={t.color}/>
          </div>
        );
      })}
      {/* [3] Editable diary note — always editable for any date */}
      <div style={{marginTop:16}}>
        <div style={{display:"flex",justifyContent:"space-between",alignItems:"center",marginBottom:8}}>
          <div style={{fontFamily:"'Orbitron',sans-serif",fontSize:11,color:C.purple}}>✍ 日記反省</div>
          {!isToday&&<Badge label="補寫模式" color={C.gold}/>}
        </div>
        <Inp multiline value={noteVal} onChange={e=>setNoteVal(e.target.value)}
          placeholder="記錄心得、反省、學習..." rows={4}/>
        <div style={{display:"flex",justifyContent:"flex-end",marginTop:8,gap:8,alignItems:"center"}}>
          {saved&&<span style={{color:C.green,fontSize:12,fontFamily:"'Share Tech Mono',monospace"}}>✅ 已儲存</span>}
          <NeonBtn small color={C.purple} onClick={saveNote}>儲存日記</NeonBtn>
        </div>
      </div>
    </Modal>
  );
}

// ══════════════════════════════════════════════
// PAGE 4 — QUESTS
// ══════════════════════════════════════════════
function QuestPage({quests,setQuests,talents,habits,setHabits}) {
  const [addOpen,setAddOpen]=useState(false);
  const [selected,setSelected]=useState(null);
  const [form,setForm]=useState({name:"",desc:"",talentId:"work",deadline:"",targetScore:100});

  const addQuest=()=>{
    if(!form.name.trim())return;
    setQuests(qs=>[...qs,{
      id:Date.now().toString(),...form,
      targetScore:Number(form.targetScore)||100,
      currentScore:0,status:"active",subs:[],
    }]);
    setForm({name:"",desc:"",talentId:"work",deadline:"",targetScore:100});
    setAddOpen(false);
  };

  const daysLeft=d=>{if(!d)return null;const diff=Math.ceil((new Date(d)-new Date())/86400000);return diff>=0?`${diff}天後`:`逾期${-diff}天`;};
  const grouped=TALENTS.map(t=>({talent:t,qs:quests.filter(q=>q.talentId===t.id)})).filter(g=>g.qs.length>0);

  return (
    <div style={{padding:16}}>
      <div style={{display:"flex",justifyContent:"space-between",alignItems:"center",marginBottom:14}}>
        <div style={{fontFamily:"'Orbitron',sans-serif",fontSize:15,color:C.neon}}>主線任務管理</div>
        <NeonBtn small onClick={()=>setAddOpen(!addOpen)}>+ 新增</NeonBtn>
      </div>
      {addOpen&&(
        <GlassCard style={{padding:14,marginBottom:14}}>
          <div style={{display:"flex",flexDirection:"column",gap:10}}>
            <Inp value={form.name} onChange={e=>setForm(f=>({...f,name:e.target.value}))} placeholder="任務名稱（必填）"/>
            <Inp value={form.desc} onChange={e=>setForm(f=>({...f,desc:e.target.value}))} placeholder="描述 / 備註"/>
            <div style={{display:"grid",gridTemplateColumns:"1fr 1fr",gap:10}}>
              <Sel value={form.talentId} onChange={e=>setForm(f=>({...f,talentId:e.target.value}))}>
                {TALENTS.map(t=><option key={t.id} value={t.id}>{t.icon} {t.name}</option>)}
              </Sel>
              <div>
                <div style={{color:C.dim,fontSize:11,marginBottom:4}}>🎯 目標分數</div>
                <Inp value={form.targetScore} onChange={e=>setForm(f=>({...f,targetScore:e.target.value}))} placeholder="100"/>
              </div>
            </div>
            <Inp value={form.deadline} onChange={e=>setForm(f=>({...f,deadline:e.target.value}))} placeholder="截止日 YYYY-MM-DD"/>
            <div style={{display:"flex",gap:10}}>
              <NeonBtn onClick={addQuest} color={C.green}>新增任務</NeonBtn>
              <NeonBtn onClick={()=>setAddOpen(false)} color={C.dim}>取消</NeonBtn>
            </div>
          </div>
        </GlassCard>
      )}
      {quests.length===0&&(
        <div style={{color:C.dim,textAlign:"center",padding:40}}>尚無主線任務<br/><span style={{fontSize:12}}>點擊「+ 新增」建立你的第一個目標</span></div>
      )}
      {grouped.map(({talent,qs})=>(
        <div key={talent.id} style={{marginBottom:16}}>
          <div style={{display:"flex",alignItems:"center",gap:8,marginBottom:8}}>
            <span>{talent.icon}</span>
            <span style={{fontFamily:"'Orbitron',sans-serif",fontSize:12,color:talent.color}}>{talent.name}</span>
          </div>
          {qs.map(q=>{
            const dl=daysLeft(q.deadline);
            const prog=Math.min(100,((q.currentScore||0)/(q.targetScore||100))*100);
            return (
              <GlassCard key={q.id} neonColor={talent.color} style={{padding:14,marginBottom:8}} onClick={()=>setSelected(q)}>
                <div style={{display:"flex",justifyContent:"space-between",alignItems:"flex-start"}}>
                  <div style={{flex:1}}>
                    <div style={{fontWeight:700,color:"#fff",fontSize:15,marginBottom:6}}>{q.name}</div>
                    <div style={{display:"flex",gap:6,flexWrap:"wrap",marginBottom:8}}>
                      <Badge label={q.status==="done"?"🟢 已完成":q.currentScore>0?"🟡 進行中":"🔴 未開始"} color={q.status==="done"?C.green:q.currentScore>0?C.gold:C.red}/>
                      {dl&&<Badge label={`⏳ ${dl}`} color={dl.includes("逾期")?C.red:C.dim}/>}
                      <Badge label={`🎯 ${q.currentScore||0}/${q.targetScore||100}`} color={talent.color}/>
                    </div>
                    <XpBar current={q.currentScore||0} needed={q.targetScore||100} color={talent.color}/>
                    <div style={{fontSize:10,color:C.dim,marginTop:4,fontFamily:"'Share Tech Mono',monospace"}}>
                      進度：{prog.toFixed(0)}%
                    </div>
                  </div>
                  <button onClick={e=>{e.stopPropagation();setQuests(qs=>qs.filter(x=>x.id!==q.id));}} style={{background:"none",border:"none",color:C.dim,cursor:"pointer",fontSize:16,marginLeft:8}}>✕</button>
                </div>
                <div style={{display:"flex",gap:8,marginTop:10}} onClick={e=>e.stopPropagation()}>
                  <NeonBtn small color={C.green} onClick={()=>setQuests(qs=>qs.map(x=>x.id===q.id?{...x,status:"done"}:x))}>標記完成</NeonBtn>
                  {q.status==="done"&&<NeonBtn small color={C.gold} onClick={()=>setQuests(qs=>qs.map(x=>x.id===q.id?{...x,status:"active"}:x))}>重新開啟</NeonBtn>}
                </div>
              </GlassCard>
            );
          })}
        </div>
      ))}
      {selected&&<QuestDetailModal quest={selected} quests={quests} setQuests={setQuests} onClose={()=>setSelected(null)}/>}
    </div>
  );
}

function QuestDetailModal({quest,quests,setQuests,onClose}) {
  const talent=TALENTS.find(t=>t.id===quest.talentId)||TALENTS[0];
  const q=quests.find(x=>x.id===quest.id)||quest;
  const prog=Math.min(100,((q.currentScore||0)/(q.targetScore||100))*100);

  // Manual score adjustment
  const [adj,setAdj]=useState("");
  const applyAdj=()=>{
    const val=parseInt(adj,10);
    if(isNaN(val))return;
    setQuests(qs=>qs.map(qq=>{
      if(qq.id!==q.id)return qq;
      const newScore=Math.max(0,(qq.currentScore||0)+val);
      const isDone=newScore>=(qq.targetScore||100);
      return {...qq,currentScore:newScore,status:isDone?"done":qq.status};
    }));
    setAdj("");
  };

  return (
    <Modal title={`📋 ${q.name}`} onClose={onClose} width={500}>
      {q.desc&&<div style={{color:C.dim,fontSize:14,marginBottom:14}}>{q.desc}</div>}

      {/* Score progress */}
      <GlassCard neonColor={talent.color} style={{padding:14,marginBottom:14}}>
        <div style={{display:"flex",justifyContent:"space-between",alignItems:"center",marginBottom:8}}>
          <div style={{fontFamily:"'Orbitron',sans-serif",fontSize:13,color:talent.color}}>🎯 目標分數進度</div>
          <div style={{fontFamily:"'Share Tech Mono',monospace",fontSize:14,color:"#fff"}}>{q.currentScore||0} / {q.targetScore||100}</div>
        </div>
        <XpBar current={q.currentScore||0} needed={q.targetScore||100} color={talent.color} height={12}/>
        <div style={{textAlign:"right",fontSize:11,color:C.dim,marginTop:4,fontFamily:"'Share Tech Mono',monospace"}}>{prog.toFixed(1)}%</div>

        {/* Manual score adjust */}
        <div style={{display:"flex",gap:8,marginTop:12,alignItems:"center"}}>
          <div style={{color:C.dim,fontSize:12,whiteSpace:"nowrap"}}>手動加分：</div>
          <Inp value={adj} onChange={e=>setAdj(e.target.value)} placeholder="+10 或 -5" style={{flex:1}}/>
          <NeonBtn small color={talent.color} onClick={applyAdj}>套用</NeonBtn>
        </div>
        <div style={{fontSize:11,color:C.dim,marginTop:6}}>💡 完成每日習慣/任務並連結此目標時，XP 會自動累積到分數</div>
      </GlassCard>

      <div style={{display:"flex",gap:8,flexWrap:"wrap"}}>
        {q.deadline&&<Badge label={`📅 ${q.deadline}`} color={C.dim}/>}
        <Badge label={q.status==="done"?"🟢 已完成":q.currentScore>0?"🟡 進行中":"🔴 未開始"} color={q.status==="done"?C.green:q.currentScore>0?C.gold:C.red}/>
      </div>
    </Modal>
  );
}

// ══════════════════════════════════════════════
// PAGE 5 — SETTINGS  [4] removed storage block
// ══════════════════════════════════════════════
function SettingsPage({char,setChar,talents,setTalents,setHabits,setQuests,setDailyRec}) {
  const [name,setName]=useState(char.name);
  const [motto,setMotto]=useState(char.motto);
  const [msg,setMsg]=useState("");
  const flash=m=>{setMsg(m);setTimeout(()=>setMsg(""),2500);};

  const exportData=()=>{
    const data={};
    for(let k in localStorage){if(k.startsWith("rpg_")){try{data[k]=JSON.parse(localStorage[k]);}catch{}}}
    const blob=new Blob([JSON.stringify(data,null,2)],{type:"application/json"});
    const a=document.createElement("a");a.href=URL.createObjectURL(blob);a.download="rpg_backup.json";a.click();
    flash("📥 資料已匯出");
  };

  const importData=e=>{
    const file=e.target.files[0];if(!file)return;
    const reader=new FileReader();
    reader.onload=ev=>{
      try{const data=JSON.parse(ev.target.result);for(let k in data)localStorage.setItem(k,JSON.stringify(data[k]));flash("✅ 已匯入，請重新整理頁面");}
      catch{flash("❌ 匯入失敗");}
    };reader.readAsText(file);
  };

  return (
    <div style={{padding:16}}>
      <div style={{fontFamily:"'Orbitron',sans-serif",fontSize:15,color:C.neon,marginBottom:16}}>⚙️ SETTINGS</div>
      {msg&&<GlassCard neonColor={C.green} style={{padding:12,marginBottom:14,textAlign:"center",color:C.green,fontFamily:"'Share Tech Mono',monospace",fontSize:14}}>{msg}</GlassCard>}

      {/* Character */}
      <GlassCard style={{padding:16,marginBottom:12}}>
        <div style={{fontFamily:"'Orbitron',sans-serif",fontSize:12,color:C.neon,marginBottom:12}}>角色設定</div>
        <div style={{display:"flex",flexDirection:"column",gap:10}}>
          <div><div style={{color:C.dim,fontSize:12,marginBottom:4}}>角色名稱</div><Inp value={name} onChange={e=>setName(e.target.value)}/></div>
          <div><div style={{color:C.dim,fontSize:12,marginBottom:4}}>格言</div><Inp value={motto} onChange={e=>setMotto(e.target.value)}/></div>
          <NeonBtn color={C.green} onClick={()=>{save("rpg_character",{name,motto});setChar({name,motto});flash("✅ 角色資料已儲存");}}>儲存角色</NeonBtn>
        </div>
      </GlassCard>

      {/* Data — export/import only, removed storage size block [4] */}
      <GlassCard style={{padding:16,marginBottom:12}}>
        <div style={{fontFamily:"'Orbitron',sans-serif",fontSize:12,color:C.neon,marginBottom:12}}>資料備份</div>
        <div style={{display:"flex",flexDirection:"column",gap:10}}>
          <NeonBtn color={C.gold} onClick={exportData}>📤 匯出資料 (JSON)</NeonBtn>
          <label style={{display:"block",cursor:"pointer"}}>
            <div style={{background:"transparent",border:`1px solid ${C.purple}`,color:C.purple,fontFamily:"'Orbitron',sans-serif",fontWeight:700,fontSize:13,letterSpacing:1,padding:"9px 20px",borderRadius:6,textTransform:"uppercase",textAlign:"center",boxShadow:`0 0 8px ${C.purple}55`}}>📥 匯入資料 (JSON)</div>
            <input type="file" accept=".json" onChange={importData} style={{display:"none"}}/>
          </label>
          <NeonBtn color={C.red} onClick={()=>{save(`rpg_daily_${todayStr()}`,{});setDailyRec({});flash("🔄 今日任務已重置");}}>🔄 重置今日任務</NeonBtn>
        </div>
      </GlassCard>

      {/* Talent overview */}
      <GlassCard style={{padding:16,marginBottom:12}}>
        <div style={{fontFamily:"'Orbitron',sans-serif",fontSize:12,color:C.neon,marginBottom:12}}>天賦 XP 概覽</div>
        {talents.map(t=>{
          const {level:lv,currentXp,neededXp}=calcLevel(t.xp||0);
          return (
            <div key={t.id} style={{marginBottom:10}}>
              <div style={{display:"flex",justifyContent:"space-between",marginBottom:4}}>
                <span style={{fontSize:13}}>{t.icon} {t.name}</span>
                <span style={{fontFamily:"'Share Tech Mono',monospace",fontSize:11,color:t.color}}>Lv.{lv} · {t.xp||0} XP</span>
              </div>
              <XpBar current={currentXp} needed={neededXp} color={t.color} height={4}/>
            </div>
          );
        })}
      </GlassCard>

      <div style={{textAlign:"center",color:C.dim,fontSize:11,fontFamily:"'Share Tech Mono',monospace",marginTop:8}}>
        RPG LIFE TRACKER v2.0 · CYBERPUNK EDITION<br/>All data stored locally · No cloud sync
      </div>
    </div>
  );
}

const root=ReactDOM.createRoot(document.getElementById("root"));
root.render(<App/>);
</script>
</body>
</html>
