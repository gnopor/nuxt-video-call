<template>
  <div class="video-call">
    <!-- button -->
    <button @click="runtest">Start Call</button>
    <!-- current user video  -->
    <video class="video-large" autoplay />
    <!-- user list  -->
    <div class="user-list">
      <h4>Users</h4>
      <div id="users" class="users" />
    </div>
    <!-- users stream container  -->
    <div id="users-container" class="users-container" />
    <div />
  </div>
</template>

<script>
import io from "socket.io-client";
export default {
  methods: {
    runtest() {
      // peerConnection
      // get and create an instance of the peerconnection
      const PeerConnection =
        window.RTCPeerConnection ||
        window.mozRTCPeerConnection ||
        window.webkitRTCPeerConnection ||
        window.msRTCPeerConnection;

      const SessionDescription =
        window.RTCSessionDescription ||
        window.mozRTCSessionDescription ||
        window.webkitRTCSessionDescription ||
        window.msRTCSessionDescription;

      navigator.getUserMedia =
        navigator.getUserMedia ||
        navigator.webkitGetUserMedia ||
        navigator.mozGetUserMedia ||
        navigator.msGetUserMedia;

      // create an instance of PeerConnection
      const pc = new PeerConnection({
        iceServers: [
          {
            url: "stun:stun.services.mozilla.com",
            username: "somename",
            credential: "somecredentials",
          },
        ],
      });

      // error handler
      function error(err) {
        console.warn("Error", err);
      }

      // define create Offer function to start peer to peer connection
      function createOffer(id) {
        pc.createOffer(function (offer) {
          pc.setLocalDescription(
            new SessionDescription(offer),
            function () {
              // make offer to the cliked id
              socket.emit("make-offer", {
                offer,
                to: id,
              });
            },
            error
          );
        }, error);
      }

      // socket
      // start connecion
      const socket = io.connect("http://localhost:5000");

      // add users list
      socket.on("add-users", (data) => {
        console.log(data);
        for (let i = 0; i < data.users.length; i++) {
          const el = document.createElement("div");
          const id = data.users[i];

          el.setAttribute("id", id);
          el.innerHTML = id;
          el.setAttribute("class", "user-item");
          el.addEventListener("click", function () {
            // create RTC peer connection when clicked
            createOffer(id);
          });
          document.getElementById("users").appendChild(el);
        }
      });

      // remove user on the current user screen
      socket.on("remove-user", function (id) {
        const div = document.getElementById(id);
        document.getElementById("users").removeChild(div);
      });

      // handle offer-made and send answer to the remote user
      socket.on("offer-made", function (data) {
        // offer = data.offer

        pc.setRemoteDescription(
          new SessionDescription(data.offer),
          function () {
            pc.createAnswer(function (answer) {
              pc.setLocalDescription(
                new SessionDescription(answer),
                function () {
                  console.log("MAKE ANSWER");
                  socket.emit("make-answer", {
                    answer,
                    to: data.socket,
                  });
                },
                error
              );
            }, error);
          },
          error
        );
      });

      // handle answer-made by setting setRemoteDescription with the remote answer object
      const answersFrom = {};
      // let offer

      socket.on("answer-made", function (data) {
        pc.setRemoteDescription(
          new SessionDescription(data.answer),
          function () {
            document
              .getElementById(data.socket)
              .setAttribute("class", "active");
            if (!answersFrom[data.socket]) {
              createOffer(data.socket);
              answersFrom[data.socket] = true;
            }
          },
          error
        );
      });

      pc.onaddstream = function (obj) {
        console.log(obj);
        const vid = document.createElement("video");
        vid.setAttribute("class", "video-small");
        vid.setAttribute("autoplay", "autoplay");
        vid.setAttribute("id", "video-small");
        document.getElementById("users-container").appendChild(vid);
        vid.srcObject = obj.stream;
      };

      navigator.getUserMedia(
        { video: true, audio: true },
        function (stream) {
          const video = document.querySelector("video");
          video.srcObject = stream;
          pc.addStream(stream);
        },
        error
      );
    },
  },
};
</script>


<style scoped>
/* .video-call  */
.video-call {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: space-evenly;
  height: 100vh;
  width: 100vw;
}
.video-call > button {
  background: transparent;
  font-size: 15px;
  font-weight: bold;
  margin: 10px 0;
  padding: 10px;
  border: 2px solid black;
  cursor: pointer;
}

.video-call > video {
  margin: auto;
  transform: rotateY(180deg);
}

/* user-list  */
.user-list {
  display: flex;
  flex-direction: column;
}

.user-list h4 {
  margin: 10px 0;
}

.user-list .users {
  display: flex;
  flex-direction: column;
  padding: 20px 0 0 0;
}

/* users-container  */
.users-container {
  background: rgba(0, 0, 0, 0.2);
  display: flex;
  justify-content: center;
  padding: 10px;
  width: 100vw;
}
</style>