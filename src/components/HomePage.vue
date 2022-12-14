<template>
    <div >
      <div v-if="!gameOver">
        <img :src = title :class="titleClass"/>
        <h2 v-if="message" class="user-message">{{message}}</h2>
        <UserMessage/>
        <div :class="formClass" >
          <form v-if="!gameId" @submit="handleCreateGame" class="start-form create">
            <input ref="createGameInput" class="input-form"/>
            <button type="submit" class="button-form"> Create Game </button>
          </form>
          <form v-if="makingName" @submit="handleAddNameToGame" class="entername">
            <div class="start-form">
              <div class="button-form">Please Enter Your Name</div>
              <input ref="addNameToGame" class="input-form"/>
              <button type="submit" class="button-form"> Enter </button>
            </div>
          </form>
          <form v-if="!gameId" @submit="handleJoinGame" class="start-form join">
            <input ref="joinGameInput" class="input-form"/>
            <button type="submit" class="button-form"> Join Game </button>
          </form>
          <a target="_blank" rel="noopener noreferrer" v-if="!gameId" class="link" href="https://www.youtube.com/watch?v=npWo5FA7aPQ">How To Play</a>
        </div>
        <LoadScene/>
        <button v-if="!gameRunning && game ? game.players.length > 1 : false && game.players[game.turnIndex].name === player.name" @click="handleStartGame()" class="start-button">START GAME</button>
        <UI v-if="gameRunning"></UI>
      </div>
      <div v-if="game">
        <GameOver v-if="game.gameOver"/>
      </div>
    </div>
  </template>
  
<script>
  import io from "socket.io-client"
  import UserMessage from "./UserMessage.vue";
  import UI from "./UI.vue"
  import LoadScene from './LoadScene.vue'
  import title from '../../public/assets/RacoionTycoonTitle_preview_rev_2.png'
  import GameOver from "./GameOver.vue";
  import { mapState, mapMutations, mapGetters } from "vuex";
  export default {
    name: 'HomePage',
    components:{
      UI,
      UserMessage,
      LoadScene,
      GameOver
    },
    data () {
      return {
        roomId: null,
        gameId: null,
        makingName: false,
        title: title,
        titleClass: "title-big",
        formClass:"start-forms"
      }
    },
    computed: {
      ...mapState(["gameRunning", "game", "name", "socket", "player", "gameCanStart", "message", "gameOver", "audio"]),
      ...mapGetters(["getGame", "getPlayer", "getSocket", "getGameRunning", "getMessage", "getGameOver", "getAudio"])
    },
    created() {
      this.updateSocket(io("https://game-test-birds-eye.herokuapp.com", { }))
      //this.updateSocket(io("http://localhost:3000", { }))
      let background = new Audio("background.mp3")
      let flip =  new Audio("flip.mp3")
      this.updateAudio({background: background, flip:flip})
    },
    watch:{
      message(){
        if(this.getMessage()){
          setTimeout(() => {
          this.updateMessage(null)
        }, 3000);
        }
      }
    },
    async mounted() {
      await this.getSocket().on('gameStarted', async data => {
          this.updateGame(data.game)
          this.updatePlayer(data.game.players.filter((player)=> player.name === this.getPlayer().name)[0])
          this.updateGameRunning(true)
          this.handleAnimateTitle()
          this.formClass = "hidden"
      })

      await this.getSocket().on("invalidRoom", ()=>{
        console.log("invalid room")
      })

      await this.getSocket().on("playerJoined", async () => {
        const data = {
          game: this.getGame(),
          roomId: this.roomId
        }
        await this.getSocket().emit("welcomePlayer", data)
      })

      await this.getSocket().on("joinedRoom", data => {
        this.updateGame(data.game)
        this.gameId = data.roomId
        this.makingName = true;
      })

      await this.getSocket().on("updateGame", data => {
        this.updateGame(data.game)
        this.updatePlayer(data.game.players.filter(player => player.name === this.getPlayer().name)[0])
      })

      await this.getSocket().on("updatePlayer", data => {
        this.updatePlayer(data.player)
      })

      await this.getSocket().on("gameStartedQuery", (data) => {
        const bool = this.getGame().turnIndex
        data.bool = bool
        this.getSocket().emit("gameStartedQueryResponse", data)
      })

      await this.getSocket().on("emitMessage", message => {
        this.updateMessage(message)
      })

      await this.getSocket().on("UPDATE_GAME", game => {

        console.log("GAME:", game)
        console.log("PLAYER:", game.players.filter(player => player.name === this.getPlayer().name)[0])
        
        let thisPlayer = game.players.filter(player => player.name === this.getPlayer().name)[0]
        
        this.updateGame(game)
        this.updatePlayer(thisPlayer)
        this.updateMessage(game.message)

        let player = this.getPlayer()
        switch(game.action){
          case "DISCARD":
            if(game.players[game.turnIndex].name === player.name){
              this.updateMessage(`You must discard ${player.commodies.length - player.commodityMax} commodies.`)
              player.discarding = true
              this.updatePlayer(player)
            }
            game.action = null
            this.updateGame(game)
            break

          case "GAME_OVER":
            this.updateMessage("Game Over!")
            this.updateGameOver(true)
            break

          default:
            break
        }
      })
    },

    methods : {
      //state methods
      ...mapMutations(["updateGameRunning", "updateGame", "updateName", "updatePlayer", "updateSocket", "updateGameCanStart", "updateMessage", "updateAudio"]),

      //pre game methods
      async handleStartGame(){
        const data = {
          game: this.getGame(),
          roomId : this.roomId
        }
        await this.getSocket().emit("startGame", data )
      },
      async handleCreateGame(e){
        e.preventDefault()
        if(this.$refs.createGameInput.value !== ""){
          this.roomId = this.$refs.createGameInput.value
          const data = {
            roomId : this.$refs.createGameInput.value
          }
          await this.getSocket().emit("createRoom", data)
          this.$refs.createGameInput.value = ""
          this.makingName = false; 
          let audio = new Audio("background.mp3")
          audio.volume = .05
          audio.loop = true
          audio.play()
          this.updateAudio({background:audio})
        }
      },
      async handleJoinGame(e){
        e.preventDefault()
        this.roomId = this.$refs.joinGameInput.value
        const data = {
          roomId : this.$refs.joinGameInput.value
        }
        await this.getSocket().emit("joinRoom", data)
        this.$refs.joinGameInput.value = ""
        this.makingName = false;
        let audio = {}
        audio.background = new Audio("background.mp3")
        audio.background.volume = .05
        audio.background.loop = true
        audio.background.play()

        audio.flip = new Audio("flip.mp3")
        audio.flip.volume=.2
        audio.flip.loop=false
        this.updateAudio(audio)

      },
      async handleAddNameToGame(e) {
        e.preventDefault()
        this.updateName(this.$refs.addNameToGame.value) 
        const data = {
          name : this.$refs.addNameToGame.value,
          roomId: this.roomId,
          game: this.getGame()
        }
        await this.getSocket().emit("addNameToGame", data)
        this.$refs.addNameToGame.value = ""
        this.makingName = false;
      },

      //animations
      handleAnimateTitle(){
          this.titleClass = "title-small"
      },
    }
  }
</script>

<style>

  @font-face {
    font-family: main;
    src: url("../../public/assets/MaguireThin-eZY8e.otf");
  }
  .fullscreen{
      position:absolute;
      bottom:0px;
      left: 50vw;
  }

  @keyframes titleAnimation{
    0% {
      position: absolute;
      width: 20vw;
      right: 40vw;
    }
    
    100% {
      position: absolute;
      width: 10vw;
      right: 85vw;
    }
  }
  .hidden{
    display: none;
    position: absolute;
    top: -100vh
  }
  .user-message{
    position:absolute;
    top:10px;
    margin: 10px;
    width:30vw;
    left: 35vw;
    font-size: 3em;
    font-family:main;
    font-weight: bolder;
    text-decoration: none ;
    color:rgb(167, 142, 119);
    padding: 15px;
    text-shadow: rgb(0, 0, 0) 2px 2px 2px;
  }
  .link{
    display: flex;
    flex-flow:row nowrap;
    margin: 10px;
    justify-content: center;
    align-items: center;
    background: rgba(43, 17, 17, 0.839);
    border:3px solid rgb(148, 98, 47);
    border-radius:50px;
    font-size: 2em;
    font-family:main;
    background: none;
    border:none;
    font-weight: bolder;
    text-decoration: none ;
    color:rgb(167, 142, 119);
    padding: 15px;
    padding-left:25px;
    padding-right:25px;
    border-radius: 20px;
    text-shadow: rgb(0, 0, 0) 2px 2px 2px;
  }
  .title-big{
    position: absolute;
    width: 20vw;
    right:40vw;
    min-width: 100px;
  }

  .title-small{
    position: absolute;
    width: 10vw;
    right: 85vw;
    animation: titleAnimation .5s ;
    min-width: 100px;
  }

  canvas{
    width: 100%;
    border: none;
    outline: none;
  }

  .flexrow{
    display: flex ; 
    flex: row nowrap;
  }
  .flex-item{
    margin: 10px;
  }

  .start-forms{
    font-size:1.5em;
    position:absolute;
    right: 2vw;
    width: 40vw;
    height: 10vh;
    display: flex;
    flex-flow: column nowrap;
    align-items:space-between;
    top:60vh;
    justify-content: end;
  }

  .start-form{
    display: flex;
    flex-flow:row nowrap;
    margin: 20px;
    justify-content: space-between;
    align-items: space-between;
    background: rgba(43, 17, 17, 0.839);
    border:3px solid rgb(148, 98, 47);
    border-radius:50px;
  }

  .button-form{
    font-size: 1.5em;
    font-family:main;
    background: none;
    border:none;
    font-weight: bolder;
    text-decoration: none ;
    color:rgb(167, 142, 119);
    padding: 15px;
    padding-left:25px;
    padding-right:25px;
    border-radius: 20px;
    text-shadow: rgb(0, 0, 0) 2px 2px 2px;
  }

  .input-form{
    font-size: 1em;
    font-family:main;
    outline:none;
    border:3px solid rgb(148, 98, 47);
    border-radius:50px;
    font-weight: bold;
    text-decoration: none ;
    color:rgb(148, 108, 70);
    padding:10px;
    padding-left:10px;
    margin:15px;
    background: rgba(43, 17, 17, 0.031);
    box-shadow: rgb(12, 12, 12) 4px 4px 10px;
    align-self : center;
    width:50%;
  }

  .entername{
    display:flex;
    flex-flow:column nowrap;
  }

  .button-form:hover{
    text-shadow: rgb(5, 4, 3) 5px 0px 5px;
  }

  .start-form:hover{
    background: rgb(43, 17, 17);
  }

  .link:hover{
    text-shadow: rgb(5, 4, 3) 6px 6px 13px;
  }

  .start-button{
    font-size: 3em;
    font-family: main;
    background: rgba(43, 17, 17, 0.839);
    border:3px solid rgb(148, 98, 47);
    font-weight: bolder;
    text-decoration: none ;
    color:rgb(167, 142, 119);
    padding: 15px;
    padding-left:25px;
    padding-right:25px;
    border-radius: 20px;
    text-shadow: rgb(0, 0, 0) 2px 2px 2px;
    margin: 15px;
  }

  .start-button:hover{
    box-shadow: 3px 3px 10px black;
  }
</style>
  