---
title:  "Realtime with Rails "
date:   2015-08-29 10:18:00
description: Server sent events and Postgres notify/listen
---

I spent the last few days trying to find a way to build a real-time feel to my [Teamboard][teamboard] project. I was mainly looking towards websocket solutions until I stumbled upon a Rails 4 new addition -- [ActionController::Live::SSE][sse] which lets you stream data to a client.

Teamboard is a project of mine where users can create white boards, add and share notes with their team. I wanted the users to be able to see the changes made to the board in real time. My solution is inspired from a couple blogs mainly [TenderLove's Is It Live?][isitlive] and [ngauthier.com][ngaut].

First, I changed the default rails server (WebRick) to Puma. Add the Puma gem to your Gemfile and check out the heroku puma docs to learn how to set it up

{% highlight ruby %} gem 'puma' {% endhighlight %}

Then I created a new route for my board controller and called my new controller method "Events". Clients can then connect to /api/board/:id/events through EventSource and listen to changes in the board.

{% highlight ruby %}
def events
  @board = current_user.boards.find(params[:id])
  response.headers['Content-Type'] = 'text/event-stream'
  sse = SSE.new(response.stream, event: "changed")
  @board.on_board_change do |change|
    change = JSON.parse(change)
    sse.write({change: change})
  end
end
{% endhighlight %}

As you can see, we're waiting for on_board_change to return a change. This would usually be a blocking operation but since we're using Puma we will be able to serve other requests concurrently. Lets check out the on_board_change method which is in the Board model

{% highlight ruby %}
after_save :notify_board_change after_create :notify_board_change

def notify_board_change(change = {:board => true})
  ActiveRecord::Base.connection_pool.with_connection do |connection|
    conn = connection.instance_variable_get(:@connection)
    conn.async_exec "NOTIFY #{channel}, '#{change.to_json}'"
  end
end

def on_board_change
 ActiveRecord::Base.connection_pool.with_connection do |connection|
   conn = connection.instance_variable_get(:@connection)
   begin
     $redis.lpush("opened_board_ids", self.id)
     conn.async_exec "LISTEN #{channel}"
     logger.info 'Starting Listening on channel #{channel}'
     loop do conn.wait_for_notify do |event, pid, change|
       yield change
     end
   end
   ensure
     conn.async_exec "UNLISTEN #{channel}"
     $redis.lrem("opened_board_ids", -1, self.id)
     logger.info 'Finished Listening on channel #{channel}'
    end
  end
end

private
def channel
  "boards_#{id}"
end
{% endhighlight %}

A lot of things are going on here. I'm choosing to use postgres's Listen/Notify which is similar to redis pub/sub. Everytime a client connects to /events, on_board_change method is called and we wait for a notify on the channel. When the board is changed, the notify_board_change method is called and we return this change to the controller. I kept every board we listen to in redis, you'll see why later. I chose to name my channels boards_:id to keep things simple later. I then use the after_save callback to NOTIFY these channels. Each Board has many BoardItems. I added callbacks to BoardItem.rb too. They call the same method as the board does as you can see here

{% highlight ruby %}
after_create :notify_board_change
after_destroy :notify_board_change
after_save :notify_board_change

def notify_board_change
  self.board.notify_board_change({:board_item => self.id})
end
{% endhighlight %}

Client side now. I'm using AngularJS in my project, here's how i connect to my API using EventSource.

{% highlight javascript %}
$timeout( () -> source = new EventSource('/api/boards/' + $routeParams.board_id + '/events')
$scope.$on '$destroy', () -> source.close()

source.addEventListener 'changed', (e) ->
  data = JSON.parse(e.data).change
  if data.board_item
    $scope.$broadcast('update-'+data.board_item)
  else if data.hb # DO NOTHING
  else ... {% endhighlight %}

Now everything works great but I found out a strange issue with the Rails Live Streaming. It seems SSE never realises when the connection is over with the client (https://github.com/rails/rails/issues/10989). This means a refresh, closing the browser and navigating away doesn't kill the thread that is waiting for changes. This is bad and makes Puma crash pretty quickly on a small number of workers and threads. There doesnt seem to be a nice way to fix this for now but I coded I workaround that is often proposed for this issue, that is a heartbeat thread, that checks if the connection is open every x seconds. I placed this in the puma initializer file puma.rb

{% highlight ruby %}
def create_heartbeat
  heartbeat = Thread.new do begin hb = {"hb": true}
  ActiveRecord::Base.connection_pool.with_connection do |connection|
    conn = connection.instance_variable_get(:@connection)
    loop do
      opened_boards = $redis.lrange("opened_board_ids", 0, -1)
      opened_boards.each do |board_id|
        conn.async_exec "NOTIFY boards_#{board_id}, '#{hb.to_json}'"
      end
      sleep 20.seconds
    end
  end
  ensure
  end
end
{% endhighlight %}

The amount of time to sleep between heartbeats is kind of a tradeoff. The longer it is, the longer a thread will be stuck before we kill it and reuse it but you probably dont want to set that number too low either. Since I kept the boards that were listened to in redis, I heartbeat all those channels and remove them from redis when they are killed

Hope this will help some of you. Sever Side Events are pretty cool but I am wondering if I should try the same with WebSockets until they fix the issue I linked above!

[teamboard]: http://teamboardapp.com
[sse]:       http://api.rubyonrails.org/classes/ActionController/Live/SSE.html
[isitlive]:  http://tenderlovemaking.com/2012/07/30/is-it-live.html
[ngaut]:     http://ngauthier.com
