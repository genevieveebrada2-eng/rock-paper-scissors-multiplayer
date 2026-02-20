<!DOCTYPE html>
<html lang="en">
<head>
<meta name="viewport" content="width=device-width, initial-scale=1">
<title>Rock Paper Scissors – Ultimate Multiplayer</title>
<link rel="manifest" href="manifest.json">
<style>
  body { font-family: Arial, sans-serif; background: linear-gradient(135deg,#ffd6e8,#e0f2fe); text-align:center; padding:20px; }
  .card { background:white; border-radius:20px; padding:20px; max-width:400px; margin:20px auto; box-shadow:0 10px 20px rgba(0,0,0,0.15); }
  h1,h2 { color:#ff69b4; }
  input,button { padding:10px; margin:5px; border-radius:12px; border:none; font-size:16px; }
  button { background:#ffb3d9; cursor:pointer; }
  #chat { height:120px; overflow-y:auto; background:#fdf2f8; padding:10px; border-radius:10px; text-align:left; margin-bottom:10px; }
  #emojiReaction { font-size:40px; margin:10px; height:50px; }
  #avatar { width:80px; height:80px; border-radius:50%; margin-top:10px; }
  #leaderboard { text-align:left; margin-top:20px; }
  #leaderboard h3 { text-align:center; }
  #emojiPicker button { font-size:20px; margin:2px; }
  canvas { position:fixed; pointer-events:none; top:0; left:0; width:100%; height:100%; }
</style>
</head>
<body>

<div class="card" id="joinBox">
  <h1>✊🏻🖐🏻✌🏻 ROCK PAPER SCISSORS</h1>
  <input id="name" placeholder="Your name"><br>
  <input id="room" placeholder="Room code (same for both players)"><br>
  <button onclick="startAuth()">Join Room</button>
  <button onclick="playBot()">Play Bot 🤖</button>
  <button onclick="rankedMatch()">🏆 Ranked Match</button>
  <button onclick="watchRoomPrompt()">👀 Spectator</button>
  <p id="rank">Rank: Bronze</p>
</div>

<div class="card" id="gameBox" style="display:none;">
  <h2 id="status">Waiting...</h2>
  <p>Wins: <span id="wins">0</span> | Losses: <span id="losses">0</span> | Draws: <span id="draws">0</span></p>

  <div id="emojiReaction"></div>
  <h3>🧑‍🎨 Avatar</h3>
  <input type="file" accept="image/*" onchange="uploadAvatar(event)">
  <img id="avatar" src="">

  <div>
    <button onclick="play('rock')">✊🏻</button>
    <button onclick="play('paper')">🖐🏻</button>
    <button onclick="play('scissors')">✌🏻</button>
    <button onclick="rematch()">🎮 Rematch</button>
  </div>

  <h3>Chat 💬</h3>
  <div id="chat"></div>
  <input id="msg" placeholder="Type message...">
  <button onclick="sendMsg()">Send</button>

  <div id="emojiPicker">
    <button onclick="sendEmoji('😀')">😀</button>
    <button onclick="sendEmoji('😂')">😂</button>
    <button onclick="sendEmoji('🔥')">🔥</button>
    <button onclick="sendEmoji('💥')">💥</button>
  </div>

  <div id="leaderboard" class="card">
    <h3>🏆 Global Leaderboard</h3>
    <div id="leaderboardList"></div>
  </div>
</div>

<!-- Firebase -->
<script src="https://www.gstatic.com/firebasejs/8.10.0/firebase-app.js"></script>
<script src="https://www.gstatic.com/firebasejs/8.10.0/firebase-auth.js"></script>
<script src="https://www.gstatic.com/firebasejs/8.10.0/firebase-database.js"></script>
<script src="https://www.gstatic.com/firebasejs/8.10.0/firebase-storage.js"></script>

<!-- Confetti -->
<script src="https://cdn.jsdelivr.net/npm/canvas-confetti@1.6.0/dist/confetti.browser.min.js"></script>

<!-- Sounds -->
<audio id="winSound" src="https://assets.codepen.io/4559253/win.mp3"></audio>
<audio id="loseSound" src="https://assets.codepen.io/4559253/lose.mp3"></audio>
<audio id="drawSound" src="https://assets.codepen.io/4559253/draw.mp3"></audio>
<audio id="clickSound" src="https://assets.codepen.io/4559253/click.mp3"></audio>

<script>
const firebaseConfig = {
  apiKey: "AIzaSyAfjWHdqOA2gvJbo89KUu80CMjFVZFnAa8",
  authDomain: "nera-xien.firebaseapp.com",
  projectId: "nera-xien",
  storageBucket: "nera-xien.firebasestorage.app",
  messagingSenderId: "403251362936",
  appId: "1:403251362936:web:dfba26db97da4bb21fc18d"
};
firebase.initializeApp(firebaseConfig);
const db = firebase.database();
const storage = firebase.storage();

let roomCode, playerId, playerName;
let wins=0, losses=0, draws=0, botMode=false;
let roundTimer;

// Sounds helper
function playSound(id){document.getElementById(id).play();}

// ⚡ Save/load progress offline
function loadProgress(){
  wins = Number(localStorage.getItem("wins")||0);
  losses = Number(localStorage.getItem("losses")||0);
  draws = Number(localStorage.getItem("draws")||0);
  playerName = localStorage.getItem("playerName")||"Player";
  document.getElementById("wins").innerText=wins;
  document.getElementById("losses").innerText=losses;
  document.getElementById("draws").innerText=draws;
}
function saveProgress(){
  localStorage.setItem("wins",wins);
  localStorage.setItem("losses",losses);
  localStorage.setItem("draws",draws);
  localStorage.setItem("playerName",playerName);
}

// ⚡ Auth
function startAuth(){
  const email=prompt("Enter email:");
  const password=prompt("Enter password:");
  firebase.auth().signInWithEmailAndPassword(email,password)
    .catch(err=>{
      if(err.code==="auth/user-not-found"){
        firebase.auth().createUserWithEmailAndPassword(email,password)
          .catch(e=>alert("Auth error: "+e.message));
      } else alert("Auth error: "+err.message);
    });
}

// ✅ Listen to auth state
firebase.auth().onAuthStateChanged(user=>{
  if(user){
    playerId = user.uid;
    console.log("Logged in as", user.email);
    // Now we can join room safely
    loadProgress();
  }
});

// ⚡ Join Room (call after auth)
function joinRoom(){
  playerName=document.getElementById("name").value||playerName;
  roomCode=document.getElementById("room").value||roomCode;
  if(!roomCode){alert("Enter room code!"); return;}
  db.ref("rooms/"+roomCode+"/players/"+playerId).set({name:playerName, move:"", wins:wins, losses:losses});
  document.getElementById("joinBox").style.display="none";
  document.getElementById("gameBox").style.display="block";
  loadAvatar();
  listenMoves();
  listenChat();
  updateLeaderboard();
  setupFriendOnlineNotification();
}

// ⚡ Avatar
function uploadAvatar(event){
  const file=event.target.files[0];
  const ref=storage.ref("avatars/"+playerId+".png");
  ref.put(file).then(()=>{
    ref.getDownloadURL().then(url=>{
      db.ref("users/"+playerId+"/avatar").set(url);
      document.getElementById("avatar").src=url;
    });
  });
}
function loadAvatar(){
  db.ref("users/"+playerId+"/avatar").once("value").then(snap=>{
    if(snap.exists()) document.getElementById("avatar").src=snap.val();
  });
}

// ⚡ Moves & Chat
function listenMoves(){
  db.ref("rooms/"+roomCode).on("value",snap=>{
    const data=snap.val();
    if(!data||!data.players) return;
    const ids=Object.keys(data.players);
    if(ids.length===2){
      const p1=data.players[ids[0]];
      const p2=data.players[ids[1]];
      if(p1.move && p2.move) handleResult(p1.move,p2.move);
    }
  });
}
function listenChat(){
  db.ref("rooms/"+roomCode+"/chat").on("child_added",snap=>{
    document.getElementById("chat").innerHTML+="<div>"+snap.val()+"</div>";
    document.getElementById("chat").scrollTop=document.getElementById("chat").scrollHeight;
  });
}

// ⚡ Game logic
function handleResult(a,b){
  const emojiMap={rock:"✊🏻",paper:"🖐🏻",scissors:"✌🏻"};
  document.getElementById("emojiReaction").innerText=emojiMap[a]+" vs "+emojiMap[b];
  confetti({particleCount:30,spread:60,startVelocity:30,origin:{y:0.4}});
  if(a===b){draws++;document.getElementById("status").innerText="Draw!";playSound('drawSound');}
  else if((a==="rock"&&b==="scissors")||(a==="paper"&&b==="rock")||(a==="scissors"&&b==="paper")){
    wins++;document.getElementById("status").innerText="You win!";playSound('winSound');confettiWin();
  } else {losses++;document.getElementById("status").innerText="You lose!";playSound('loseSound');}
  document.getElementById("wins").innerText=wins;
  document.getElementById("losses").innerText=losses;
  document.getElementById("draws").innerText=draws;
  saveProgress();
  db.ref("users/"+playerId+"/stats").set({wins:wins,losses:losses,draws:draws,name:playerName});
  db.ref("rooms/"+roomCode+"/players").once("value").then(snap=>{
    snap.forEach(p=>db.ref("rooms/"+roomCode+"/players/"+p.key+"/move").set(""));
  });
  updateLeaderboard();
}

function play(choice){
  playSound('clickSound');
  const emojiMap={rock:"✊🏻",paper:"🖐🏻",scissors:"✌🏻"};
  document.getElementById("emojiReaction").innerText=emojiMap[choice];
  confetti({particleCount:20,spread:50,origin:{y:0.3}});
  if(botMode){handleResult(choice, botChoiceAI());}
  else db.ref("rooms/"+roomCode+"/players/"+playerId+"/move").set(choice);
}

function rematch(){document.getElementById("status").innerText="Waiting for rematch...";db.ref("rooms/"+roomCode+"/players/"+playerId+"/move").set("");}

function sendMsg(){const msg=document.getElementById("msg").value;if(!msg)return;playSound('clickSound');db.ref("rooms/"+roomCode+"/chat").push(playerName+": "+msg);document.getElementById("msg").value="";}

function sendEmoji(emoji){db.ref("rooms/"+roomCode+"/chat").push(playerName+": "+emoji);}

function confettiWin(){confetti({particleCount:100,spread:90,origin:{y:0.6}});}

// ⚡ Leaderboard
function updateLeaderboard(){
  db.ref("users").orderByChild("stats/wins").limitToLast(10).on("value",snap=>{
    const list=document.getElementById("leaderboardList");
    list.innerHTML="";
    const sorted=Object.entries(snap.val()||{}).sort((a,b)=>b[1].stats.wins - a[1].stats.wins);
    sorted.forEach(([id,data],i)=>{
      const avatar=data.avatar||"";
      const name=data.stats?.name||"Player";
      const wins=data.stats?.wins||0;
      const losses=data.stats?.losses||0;
      list.innerHTML+=`<div>${i+1}. <img src="${avatar}" width="30" height="30"> ${name} - Wins:${wins} Losses:${losses}</div>`;
    });
  });
}

// ⚡ Ranked Match logic
function rankedMatch(){
  if(!playerId){alert("Sign in first!"); return;}
  const queueRef=db.ref("rankedQueue/"+playerId);
  queueRef.set({name:playerName,wins:wins,uid:playerId});
  db.ref("rankedQueue").once("value").then(snap=>{
    const players=snap.val()||{};
    for(let id in players){
      if(id!==playerId){
        const opponent=players[id];
        if(Math.abs(opponent.wins-wins)<=5){
          const roomId="R"+Date.now();
          db.ref("rooms/"+roomId+"/players/"+playerId).set({name:playerName,move:""});
          db.ref("rooms/"+roomId+"/players/"+opponent.uid).set({name:opponent.name,move:""});
          db.ref("rankedQueue/"+playerId).remove();
          db.ref("rankedQueue/"+opponent.uid).remove();
          roomCode=roomId;
          document.getElementById("joinBox").style.display="none";
          document.getElementById("gameBox").style.display="block";
          listenMoves(); listenChat(); updateLeaderboard();
          break;
        }
      }
    }
  });
}

// ⚡ Friend Online Notification
function setupFriendOnlineNotification(){
  Notification.requestPermission();
  db.ref("friends/"+playerId).on("child_changed",snap=>{
    if(snap.val().online) new Notification(`${snap.val().name} is online!`);
  });
}

// ⚡ Spectator
function watchRoomPrompt(){
  const r=prompt("Enter room code to watch:");
  if(!r) return;
  roomCode=r;
  document.getElementById("joinBox").style.display="none";
  document.getElementById("gameBox").style.display="block";
  db.ref("rooms/"+roomCode).on("value",snap=>{
    const data=snap.val();
    if(!data||!data.players) return;
    document.getElementById("emojiReaction").innerText=Object.values(data.players).map(p=>p.move||"❓").join(" vs ");
    document.getElementById("status").innerText="Spectating...";
  });
}

// ⚡ AI Bot
function botChoiceAI(){
  const choices=["rock","paper","scissors"];
  return choices[Math.floor(Math.random()*3)];
}
function playBot(){botMode=true;document.getElementById("joinBox").style.display="none";document.getElementById("gameBox").style.display="block";document.getElementById("status").innerText="Playing Bot 🤖";}
</script>
</body>
</html>
