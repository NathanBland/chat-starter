<template lang='jade'>
  .room
    ul.breadcrumb
      li.breadcrumb-item
        a(v-link="'/home'") Home
      li.breadcrumb-item
          a(v-link="{ name: 'room', params: { room: $route.params.room }}") {{$route.params.room}}
    user-chip(:users='users')
    message-list(:messages='messages')
    message-input
</template>


<script>
  import userChip from './UserChip'
  import messageList from './messageList'
  import messageInput from './messageInput'
  import auth from '../assets/auth'
  import {router} from '../main'
  import io from 'socket.io-client'
  
  import Lockr from 'lockr'
  export default {
    components: {
      'messageList': messageList,
      'messageInput': messageInput,
      'userChip': userChip},
    data () {
      return {
        messages: [],
        users: [],
        connected: false,
        socket: {}
      }
    },
    ready: function () {
      let myVue = this
      var socket = io.connect('//resound-api.herokuapp.com', {
        query: 'token=' + Lockr.get('token')
      })
      this.$set('socket', socket)
      socket.on('rejected', function (data) {
        console.log('we must have been bad. :(')
      })
      socket.on('authenticated', function (data) {
        console.log('Yay us!')
      })
      socket.on('joined', function (data) {
        console.log('joined room:', data)
      })
      socket.on('message', function (data) {
        myVue.messages.push({user: data.user, msg: data.msg})
      })
      this.$http({url: 'room/' + this.$route.params.room, method: 'GET'}).then(function (res) {
        console.log('exists res:', res)
        this.$set('messages', res.data.messages)
        this.$set('users', res.data.users)
      }, function (res) {
        console.log('Room isnt a thing?:', res)
        if (!res.ok) {
          console.log('trying to create room')
          this.$http({
            url: 'room',
            method: 'POST',
            data: {title: this.$route.params.room}
          }).then(function (res) {
            console.log('success:', res)
            router.go('/room/' + res.data.alias)
            this.$set('messages', res.data.messages)
            this.$set('users', res.data.users)
            socket.on('connect', function (data) {
                // console.log('got Socket Connection:', data)
              socket.emit('join', {room: res.data.alias})
            })
          }, function (res) {
            console.log('Room is already a thing!:', res)
            router.go('/room/' + res.data.room.alias)
            this.$set('messages', res.data.room.messages)
            this.$set('users', res.data.room.users)
            socket.on('connect', function (data) {
            // console.log('got Socket Connection:', data)
              socket.emit('join', {room: res.data.room.alias})
            })
          })
        }
      })
    },
    events: {
      sendMessage (message) {
        // console.log('(message) auth:', auth.user)
        if (auth.user.username) {
          this.messages.push({user: auth.user.username, msg: message})
          this.socket.emit('message', message)
          return true
        }
        return false
      }
    },
    route: {
      canActivate () {
        return auth.user.authenticated
      },
      canReuse () {
        return false
      }
    },
    props: ['room']
  }
</script>

<style scoped lang='sass'>
.room
  display: flex
  flex-flow: column nowrap
  height: 89vh
</style>
