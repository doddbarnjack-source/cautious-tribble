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
