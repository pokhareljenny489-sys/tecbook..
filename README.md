<!DOCTYPE html>
<html>
<head>
<meta charset="UTF-8">
<title>TecBook</title>
<style>
body { margin:0; font-family: Arial,sans-serif; background:#f0f2f5; }
header { background:#1877f2; color:white; padding:10px; font-size:24px; font-weight:bold; }
#loginPage,#mainPage { display:none; height:100vh; }
#loginBox { width:300px; margin:100px auto; background:white; padding:20px; border-radius:8px; box-shadow:0 2px 4px rgba(0,0,0,0.2); }
#loginBox input { width:100%; padding:10px; margin:10px 0; }
#loginBox button { width:100%; padding:10px; background:#1877f2; color:white; border:none; border-radius:4px; font-size:16px; cursor:pointer; }

#mainLayout { display:flex; height:calc(100vh - 50px); }
#sidebar { width:200px; background:white; padding:10px; border-right:1px solid #ccc; overflow-y:auto; }
#feed { flex:1; padding:20px; overflow-y:auto; }
#postBox { background:white; padding:10px; border-radius:8px; margin-bottom:20px; }
#postBox input { width:80%; padding:10px; }
#postBox button { padding:10px; background:#1877f2; color:white; border:none; border-radius:4px; cursor:pointer; }
.post { background:white; padding:10px; border-radius:8px; margin-bottom:10px; }
.comment { margin-left:20px; font-size:0.9em; color:gray; }

#videoCall { position:fixed; bottom:10px; right:10px; background:white; padding:10px; border-radius:8px; box-shadow:0 2px 6px rgba(0,0,0,0.3); }
video { width:150px; border:1px solid #ccc; margin:5px; }

#chatTabs { position:fixed; bottom:0; left:220px; display:flex; }
.chatWindow { width:250px; background:white; border:1px solid #ccc; border-radius:8px 8px 0 0; margin-right:10px; display:flex; flex-direction:column; }
.chatHeader { background:#1877f2; color:white; padding:5px; cursor:pointer; font-weight:bold; }
.chatMessages { height:200px; overflow-y:auto; padding:5px; flex:1; }
.chatInput { display:flex; }
.chatInput input { flex:1; padding:5px; }
.chatInput button { padding:5px; background:#1877f2; color:white; border:none; }
.msg { margin:5px 0; }
.me { color:#1877f2; font-weight:bold; }
</style>
<script src="https://unpkg.com/peerjs@1.5.4/dist/peerjs.min.js"></script>
</head>
<body>
<header>TecBook</header>

<!-- Login -->
<div id="loginPage">
  <div id="loginBox">
    <h2>Login to TecBook</h2>
    <input id="username" placeholder="Enter username">
    <button onclick="login()">Login</button>
  </div>
</div>

<!-- Main Page -->
<div id="mainPage">
  <div id="mainLayout">
    <div id="sidebar">
      <h3>Friends</h3>
      <div id="users"></div>
    </div>
    <div id="feed">
      <div id="postBox">
        <input id="postInput" placeholder="What's on your mind?">
        <button onclick="sendPost()">Post</button>
      </div>
      <div id="posts"></div>
    </div>
  </div>

  <div id="videoCall">
    <video id="localVideo" autoplay muted></video>
    <video id="remoteVideo" autoplay></video>
  </div>

  <div id="chatTabs"></div>
</div>

<script>
let peer, myName, connections={}, posts=[], chats={};
document.getElementById("loginPage").style.display="block";

function login(){
  myName=document.getElementById("username").value;
  if(!myName) return alert("Enter username!");
  document.getElementById("loginPage").style.display="none";
  document.getElementById("mainPage").style.display="block";
  peer=new Peer(null,{host:"peerjs.com",secure:true,port:443});
  peer.on("open",id=>{
    console.log("Peer ID:",id);
    addUser(id,myName);
  });
  peer.on("connection",conn=>{
    conn.on("data",data=>handleData(data,conn.peer));
    connections[conn.peer]=conn;
  });
  peer.on("call",call=>{
    navigator.mediaDevices.getUserMedia({video:true,audio:true}).then(stream=>{
      document.getElementById("localVideo").srcObject=stream;
      call.answer(stream);
      call.on("stream",remoteStream=>{
        document.getElementById("remoteVideo").srcObject=remoteStream;
      });
    });
  });
}

// Posts
function broadcast(data){ for(let id in connections) connections[id].send(data); }
function handleData(data,from){
  if(data.type==="post"){ posts.unshift(data.post); renderPosts(); }
  else if(data.type==="user"){ addUser(from,data.name); }
  else if(data.type==="chat"){ addChatMsg(from,data.user,data.text); }
}
function sendPost(){
  let text=document.getElementById("postInput").value;
  if(!text) return;
  let post={user:myName,text,likes:0,comments:[]};
  posts.unshift(post);
  renderPosts();
  broadcast({type:"post",post});
  document.getElementById("postInput").value="";
}
function renderPosts(){
  let feed=document.getElementById("posts");
  feed.innerHTML="";
  posts.forEach((p,i)=>{
    let html=`<div class="post"><b>${p.user}</b>: ${p.text}<br>
      ❤️ ${p.likes} <button onclick="likePost(${i})">Like</button>
      <button onclick="commentPost(${i})">Comment</button>`;
    p.comments.forEach(c=>{ html+=`<div class="comment">${c.user}: ${c.text}</div>`; });
    html+="</div>";
    feed.innerHTML+=html;
  });
}
function likePost(i){ posts[i].likes++; renderPosts(); broadcast({type:"post",post:posts[i]}); }
function commentPost(i){ let text=prompt("Enter comment:"); if(!text) return; posts[i].comments.push({user:myName,text}); renderPosts(); broadcast({type:"post",post:posts[i]}); }

// Users/Friends
function addUser(id,name){
  if(document.getElementById("user-"+id)) return;
  let div=document.createElement("div");
  div.id="user-"+id;
  div.innerHTML=name+` <button onclick="connectTo('${id}')">Connect</button>
  <button onclick="callUser('${id}')">Call</button>
  <button onclick="openChat('${id}','${name}')">Chat</button>`;
  document.getElementById("users").appendChild(div);
}
function connectTo(id){ if(connections[id]) return; let conn=peer.connect(id); conn.on("open",()=>{conn.send({type:"user",name:myName});}); conn.on("data",data=>handleData(data,id)); connections[id]=conn; }
function callUser(id){ navigator.mediaDevices.getUserMedia({video:true,audio:true}).then(stream=>{document.getElementById("localVideo").srcObject=stream; let call=peer.call(id,stream); call.on("stream",remoteStream=>{document.getElementById("remoteVideo").srcObject=remoteStream;});}); }

// Multi-friend Chat
function openChat(peerId,friendName){
  if(chats[peerId]) return;
  let chatDiv=document.createElement("div"); chatDiv.className="chatWindow";
  let header=document.createElement("div"); header.className="chatHeader"; header.innerText=friendName;
  header.onclick=()=>{ let msgs=chatDiv.querySelector(".chatMessages"); msgs.style.display=msgs.style.display==="none"?"block":"none"; }
  let messagesDiv=document.createElement("div"); messagesDiv.className="chatMessages";
  let inputDiv=document.createElement("div"); inputDiv.className="chatInput";
  let input=document.createElement("input");
  let sendBtn=document.createElement("button"); sendBtn.innerText="Send"; sendBtn.onclick=()=>{ sendChat(peerId,input.value); input.value=""; }
  inputDiv.appendChild(input); inputDiv.appendChild(sendBtn);
  chatDiv.appendChild(header); chatDiv.appendChild(messagesDiv); chatDiv.appendChild(inputDiv);
  document.getElementById("chatTabs").appendChild(chatDiv);
  chats[peerId]={windowDiv:chatDiv,messagesDiv:messagesDiv,input:input};
}
function sendChat(peerId,text){ if(!text) return; addChatMsg(peerId,"Me",text); if(connections[peerId]) connections[peerId].send({type:"chat",user:myName,text}); }
function addChatMsg(peerId,user,text){ if(!chats[peerId]) openChat(peerId,user==="Me"?peerId:user); let msgDiv=document.createElement("div"); msgDiv.className="msg"; msgDiv.innerHTML=`<span class="${user==='Me'?'me':''}">${user}:</span> ${text}`; chats[peerId].messagesDiv.appendChild(msgDiv); chats[peerId].messagesDiv.scrollTop=chats[peerId].messagesDiv.scrollHeight; }
</script>
</body>
</html>
