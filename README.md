<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<title>GameTalkGG</title>
<style>
body{font-family:sans-serif;background:#f0f0f0;margin:0;padding:0;}
header{background:purple;color:yellow;padding:1rem;text-align:center;font-size:1.5rem;}
.container{max-width:800px;margin:2rem auto;padding:1rem;background:#fff;border-radius:10px;}
input,button,textarea,select{padding:.5rem;margin:.5rem 0;width:100%;border-radius:5px;border:1px solid #ccc;}
.post{border-bottom:1px solid #ddd;padding:1rem 0;}
.avatar{width:40px;height:40px;border-radius:50%;vertical-align:middle;margin-right:.5rem;}
.like{color:purple;cursor:pointer;margin-left:1rem;}
.message{border-bottom:1px solid #ddd;padding:.5rem 0;}
.chat{max-height:200px;overflow-y:auto;border:1px solid #ccc;padding:.5rem;margin-bottom:.5rem;border-radius:5px;}
</style>
</head>
<body>
<header>GameTalkGG</header>
<div class="container">
<h3>Login / Register</h3>
<input id="username" placeholder="Username">
<input id="password" type="password" placeholder="Password">
<button onclick="login()">Login</button>
<button onclick="register()">Register</button>
<hr>
<h3>Upload Avatar</h3>
<input type="file" id="avatarInput">
<button onclick="uploadAvatar()">Upload Avatar</button>
<hr>
<h3>Create Post</h3>
<textarea id="postContent" placeholder="Write something..."></textarea>
<button onclick="createPost()">Post</button>
<hr>
<h3>Feed</h3>
<div id="feed"></div>
<hr>
<h3>Private Messaging</h3>
<select id="userSelect"></select>
<div class="chat" id="chatBox"></div>
<input id="chatInput" placeholder="Type a message">
<button onclick="sendMessage()">Send</button>
</div>

<script src="https://cdn.socket.io/4.7.5/socket.io.min.js"></script>
<script>
const BACKEND_URL="https://backend.gametalkgg.com"; // Replace with your backend
let token="", socket, userId=null, selectedUserId=null;

// Login
async function login(){
  const u=document.getElementById("username").value;
  const p=document.getElementById("password").value;
  const res=await fetch(BACKEND_URL+"/api/auth/login",{method:"POST",headers:{"Content-Type":"application/json"},body:JSON.stringify({username:u,password:p})});
  const data=await res.json();
  if(data.access){token=data.access;userId=data.user.id;alert("Logged in");setupSocket();loadFeed();loadUsers();}
  else alert(data.error||"Login failed");
}

// Register
async function register(){
  const u=document.getElementById("username").value;
  const p=document.getElementById("password").value;
  const res=await fetch(BACKEND_URL+"/api/auth/register",{method:"POST",headers:{"Content-Type":"application/json"},body:JSON.stringify({username:u,password:p,email:u+"@example.com"})});
  const data=await res.json();
  alert(data.message||"Registered");
}

// Avatar Upload
async function uploadAvatar(){
  const file=document.getElementById("avatarInput").files[0];
  if(!file) return;
  const form=new FormData();
  form.append("avatar",file);
  const res=await fetch(BACKEND_URL+"/api/upload/avatar",{method:"POST",headers:{Authorization:"Bearer "+token},body:form});
  const data=await res.json();
  alert(data.url?"Avatar uploaded":"Upload failed");
  loadFeed();
}

// Create Post
async function createPost(){
  const content=document.getElementById("postContent").value;
  if(!content) return;
  await fetch(BACKEND_URL+"/api/posts",{method:"POST",headers:{"Content-Type":"application/json","Authorization":"Bearer "+token},body:JSON.stringify({content})});
  document.getElementById("postContent").value="";
  loadFeed();
}

// Load Feed
async function loadFeed(){
  const res=await fetch(BACKEND_URL+"/api/posts");
  const data=await res.json();
  const feed=document.getElementById("feed"); feed.innerHTML="";
  data.posts.forEach(p=>{
    const div=document.createElement("div");
    div.className="post";
    div.innerHTML=`<img src="${p.author.avatarUrl||'https://i.pravatar.cc/40'}" class="avatar"><strong>${p.author.username}</strong>: ${p.content} <span class="like" onclick="likePost(${p.id})">❤️ ${p.likesCount}</span>`;
    feed.appendChild(div);
  });
}

// Like Post
async function likePost(id){
  await fetch(BACKEND_URL+"/api/posts/"+id+"/like",{method:"POST",headers:{Authorization:"Bearer "+token}});
  loadFeed();
}

// Load Users for Messaging
async function loadUsers(){
  const res=await fetch(BACKEND_URL+"/api/auth/users",{headers:{Authorization:"Bearer "+token}});
  const users=await res.json();
  const select=document.getElementById("userSelect");
  select.innerHTML="";
  users.filter(u=>u.id!==userId).forEach(u=>{
    const option=document.createElement("option");
    option.value=u.id; option.text=u.username;
    select.appendChild(option);
  });
  selectedUserId=select.value;
  select.onchange=()=>selectedUserId=select.value;
}

// Setup Socket.IO
function setupSocket(){
  socket=io(BACKEND_URL,{auth:{token}});
  socket.on("connect",()=>console.log("Socket connected"));
  socket.on("private_message",msg=>{
    if(msg.from===selectedUserId || msg.to===selectedUserId){
      const box=document.getElementById("chatBox");
      box.innerHTML+=`<div class="message"><strong>${msg.fromUsername===userId?'Me':msg.fromUsername}:</strong> ${msg.content}</div>`;
      box.scrollTop=box.scrollHeight;
    }
  });
}

// Send Private Message
function sendMessage(){
  const content=document.getElementById("chatInput").value;
  if(!content || !selectedUserId) return;
  socket.emit("send_private_message",{to:selectedUserId,content});
  const box=document.getElementById("chatBox");
  box.innerHTML+=`<div class="message"><strong>Me:</strong> ${content}</div>`;
  document.getElementById("chatInput").value="";
  box.scrollTop=box.scrollHeight;
}

// Initial Load
loadFeed();
</script>
</body>
</html>
// server.js snippet
const onlineUsers = new Map();
io.on("connection", socket=>{
  const userId = socket.handshake.auth.userId;
  onlineUsers.set(userId,socket.id);

  socket.on("send_private_message", async ({to, content})=>{
    // persist message in DB here
    const recipientSocket=onlineUsers.get(to);
    if(recipientSocket){
      io.to(recipientSocket).emit("private_message",{from:userId,to,content,fromUsername:userId});
    }
  });

  socket.on("disconnect",()=>onlineUsers.delete(userId));
});
