<template>
  <div class="container">
    <h1 class="text-center">Laravel Video Chat</h1>
    <div class="videos">
      <div>
        <video id="localVideo" muted playsinline></video>
        <!-- <div class="soundbar"><span class="currentVolume"></span></div> -->
      </div>
    </div>
  </div>
</template>
<script>
import Pusher from "pusher-js";
import Peer from "simple-peer";
export default {
  props: ["user", "others"],
  data() {
    return {
      localVideo: null,
      channel: null,
      socketId: null,
      localStream: null,
      connections: {},

      peerConnectionConfig: {
        iceServers: [
          { urls: "stun:stun.services.mozilla.com" },
          { urls: "stun:stun.l.google.com:19302" },
        ],
      },
    };
  },

  methods: {
    setupVideoChat() {
      this.localVideo = document.getElementById("localVideo");
      const constraints = {
        video: true,
        audio: true,
      };

      if (navigator.mediaDevices.getUserMedia) {
        navigator.mediaDevices
          .getUserMedia(constraints)
          .then(this.getUserMediaSuccess)
          .then(this.subscribeToChannel);
      } else {
        alert("Your browser does not support getUserMedia API");
      }
    },

    getUserMediaSuccess(stream) {
      this.localStream = stream;
      this.localVideo.srcObject = stream;

      this.localVideo.onloadedmetadata = (e) => {
        this.localVideo.play();
      };
    },

    createMedia(id) {
      const videos = document.querySelectorAll("video"),
        video = document.createElement("video"),
        div = document.createElement("div"),
        media = new MediaStream();

      video.setAttribute("data-socket", id);
      video.srcObject = media;
      video.autoplay = true;
      video.playsinline = true;

      div.appendChild(video);
      document.querySelector(".videos").appendChild(div);

      return media;
    },

    gotMessageFromServer(data) {
      //Parse the incoming signal
      var signal = JSON.parse(data.message);
      //Make sure it's not coming from yourself
      if (data.from !== this.socketId) {
        if (signal.sdp) {
          this.connections[data.from]
            .setRemoteDescription(new RTCSessionDescription(signal.sdp))
            .then(() => {
              if (signal.sdp.type == "offer") {
                this.connections[data.from]
                  .createAnswer()
                  .then((description) => {
                    this.connections[data.from]
                      .setLocalDescription(description)
                      .then(() => {
                        this.channel.trigger("client-signal", {
                          to: data.from,
                          message: JSON.stringify({
                            sdp: this.connections[data.from].localDescription,
                          }),
                        });
                      })
                      .catch((e) => console.log(e));
                  })
                  .catch((e) => console.log(e));
              }
            })
            .catch((e) => console.log(e));
        }

        if (signal.ice) {
          this.connections[data.from]
            .addIceCandidate(new RTCIceCandidate(signal.ice))
            .catch((e) => console.log(e));
        }
      }
    },
    subscribeToChannel: function () {
      Pusher.logToConsole = true;
      const pusher = new Pusher(process.env.MIX_PUSHER_APP_KEY, {
        cluster: process.env.MIX_PUSHER_APP_CLUSTER,
        encrypted: true,
        wsPort: process.env.MIX_LARAVEL_WEBSOCKETS_PORT,
        wssPort: process.env.MIX_LARAVEL_WEBSOCKETS_PORT,
        wsHost: window.location.hostname,
        enabledTransports: ["ws", "wss"],
        cluster: process.env.MIX_PUSHER_APP_CLUSTER,
        authEndpoint: "/broadcasting/auth",
        auth: {
          headers: {
            "X-CSRF-Token": document.head.querySelector(
              'meta[name="csrf-token"]'
            ).content,
          },
        },
      });

      this.channel = pusher.subscribe("private-video-chat");

      pusher.connection.bind('connected', () => {
        this.socketId = pusher.connection.socket_id;
      });
      
      this.channel.bind("signal", this.gotMessageFromServer);

      this.channel.bind("pusher:subscription_succeeded", (member) => {
        this.channel.bind("pusher:member_removed", (data) => {
          var video = document.querySelector(
            '[data-socket="' + data.user_id + '"]'
          );
          if (video) {
            var parentDiv = video.parentElement;
            video.parentElement.parentElement.removeChild(parentDiv);
          }
        });

        this.channel.bind("user-joined", (data) => {
          this.newMember(data);
        });
      });
    },

    newMember: function (data) {
      data.connections.forEach((channel_member) => {
        if (this.connections[`${channel_member}`] === undefined) {
          this.connections[`${channel_member}`] = new RTCPeerConnection(
            this.peerConnectionConfig
          );
          //Wait for their ice candidate
          if (channel_member !== this.socketId) {
            this.setupPeerConnection(channel_member)
          }
        }
        
      });
      //Create an offer to connect with your local description
      if (data.count >= 2) {
        this.connections[data.socket_id].createOffer().then((description) => {
          this.connections[data.socket_id]
            .setLocalDescription(description)
            .then(() => {
              this.channel.trigger("client-signal", {
                to: data.socket_id,
                message: JSON.stringify({
                  sdp: this.connections[data.socket_id].localDescription,
                }),
              });
            })
            .catch((e) => console.log(e));
        });
      }
    },

    setupPeerConnection(channel_member) {
      this.connections[`${channel_member}`].onicecandidate = (event) => {
        if (event.candidate != null) {
          this.channel.trigger("client-signal", {
            to: `${channel_member}`,
            message: JSON.stringify({ ice: event.candidate }),
          });
        }
      };

      const media = this.createMedia(channel_member);

      //Wait for their video stream
      this.connections[`${channel_member}`].ontrack = (event) => {
        media.addTrack(event.track, media);
      };

      this.localStream.getTracks().forEach((track) => {
        this.connections[`${channel_member}`].addTrack(track, this.localStream);
      });
    },
  },
  mounted() {
    this.setupVideoChat();
  },
};
</script>
<style>
.video-container {
  width: 500px;
  height: 380px;
  margin: 8px auto;
  border: 3px solid #000;
  position: relative;
  box-shadow: 1px 1px 1px #9e9e9e;
}
.video-here {
  width: 130px;
  position: absolute;
  left: 10px;
  bottom: 16px;
  border: 1px solid #000;
  border-radius: 2px;
  z-index: 2;
}
.video-there {
  width: 100%;
  height: 100%;
  z-index: 1;
}
.text-right {
  text-align: right;
}
</style>
