视频地址：http://railscasts-china.com/episodes/action-cable-rails-5


1. rails命令创建新app


    $ rails new actioncable_app

1.1 在Gemfile文件中加入gem 'jquery-rails'

1.2在app/assets/javascripts/application.js文件中加入启用jquery的注释


    //= require jquery
    //= require rails-ujs
    
2. rails命令创建controller


    $ rails generate controller rooms show

3. 修改config/routes.rb


    root to : 'rooms#show'

4. rails命令创建model message


    $ rails generate model message content:string

5. rails命令db迁移


    $ rails db:migrate

6. 修改app/controllers/rooms_controller.rb


    def show
        @messages = Message.all
    end


7. 创建并修改app/views/messages/_message.html.erb


    <div class = "message">
        <p><%= message.content %></p>
    </div>
    
8. rails命令进入console创建新message

    
    $ rails console
    > Message.create! content: "Hello world!"
    
9. 修改app/views/rooms/show.html.erb


    <h1>Chat room</h1>
    <div id = "messages">
        <%= render @messages %>
    </div>


10. rails命令创建新channel


    $ rails generate channel room speak
    
11. 在config/routes.rb中添加actioncable服务端路径


    mount ActionCable.server => '/cable'
    
12. 打开app/assets/javascripts/cable.coffee中的@App

    
    @App ||={}
    App.cable = ActionCable.createConsumer()
    
13. 修改app/asserts/javascripts/channels/room.coffee


    speak: (message) ->
    @perform 'speak', message: message
    
14. 修改app/channels/room_channel.rb中的subscribed函数及speak函数


    class RoomChannel < ApplicationCable::Channel
        def subscribed
            stream_from "room_channel"
        end
        
        def unsubscribed
        end
        
        def speak(data)
            ActionCable.server.broadcast 'room_channel', message: data['message']
        end
    end
    
15. 修改app/asserts/javascripts/channels/room.coffee中的received方法


    received: (data) ->
        alert data['message']
        
15.1 这时打开调试服务器测试，会跳出js的alert动作

使用Chrome浏览器的检查，在console中输入：


    >App.room.speak('xixi')


16. 在app/views/rooms/show.html.erb中添加输入框，对应的数据传给room_speaker


    <h1>Chat room</h1>

    <div id="messages">
        <%= render @messages %>
    </div>
    
    <form>
        <lable>Say something:</lable><br>
        <input type="text" data-behavior="room_speaker">
    </form>

17. 在app/assets/javascripts/channels/room.coffee中添加js获取客户端输入，写在speak后面


    App.room = App.cable.subscriptions.create "RoomChannel",
      connected: ->
        # Called when the subscription is ready for use on the server
    
      disconnected: ->
        # Called when the subscription has been terminated by the server
    
      received: (data) ->
        alert data['message']
    
      speak: (message) ->
        @perform 'speak', message: message
    
    $(document).on 'keypress', '[data-behavior~=room_speaker]', (event) ->
      if event.keyCode is 13 # return = send
        App.room.speak event.target.value
        event.target.value = ''
        event.preventDefault()

18. 修改app/channels/room_channel.rb中的speak函数，把获取的message存到数据库，把广播的功能移出到后续的job

    
    def speak(data)
        Message.create! content: data['message']
    end
    
19. 修改app/models/message.rb，添加after_create_commit方法，让消息入库后传给job广播


    class Message < ApplicationRecord
        after_create_commit { MessageBroadcastJob.perform_later self }
    end

20. rails命令创建新的job文件用来广播收到的消息


    $ rails generate job MessageBroadcast

21. 修改app/jobs/message_broadcast_job.rb文件


    class MessageBroadcastJob < ApplicationJob
      queue_as :default
    
      def perform(message)
        ActionCable.server.broadcast 'room_channel', message: render_message(message)
      end
    
      private
        def render_message(message)
          ApplicationController.renderer.render(partial: 'messages/message', locals: { message: message })
        end 
    
    end

22. 修改app/assets/javascripts/channels/room.coffee中的received方法


    received: (data) ->
    $('#messages').append data['message']
    
23. 修改app/views/messages/_message.html.erb  添加cache


    <% cache message do %>
        <div class="message">
            <p><%= message.content %></p>
        </div>
    <% end %>