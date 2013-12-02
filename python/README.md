# 服务使用示例

下面我们使用[TicTacToe(井字棋)](http://zh.wikipedia.org/wiki/%E4%BA%95%E5%AD%97%E6%A3%8B)游戏来示范channel服务的使用方法：

![game-screenshot](https://github.com/sinacloud/sae-channel-examples/raw/master/python/screenshot.png)

**channel的创建和连接**

当用户A打开TicTacToe游戏的主页时，服务端的程序会：

+ 调用`create_channel`创建为用户A创建一个channel，并将该channel的url嵌入到返回给用户的html页面代码中。
+ 生成一个加入游戏的连接，用户通过将此连接发送给其它用户B，其它用户B可以通过此连接加入用户A创建的游戏。

每个页面对应的channel的name应该是独一无二的，比如可以使用用户id的字符串作为channel的name。

游戏的主页的html代码模板大致如下所示，其中`{{ url }}`和`{{ game_link }}`分别为上面生成的channel url和游戏加入连接。

    <head>
    ...
    <script src="http://channel.sinaapp.com/api.js"></script>
    </head>
    <body>
      <script>
        channel = new sae.Channel('{{ url }}');
        socket.onopen = onOpened;
        socket.onmessage = onMessage;
        socket.onerror = onError;
        socket.onclose = onClose;
      </script>

      ...

      <div id='other-player' style='display:none'>
        Waiting for another player to join.<br>
        Send them this link to play:<br>
        <div id='game-link'><a href='{{ game_link }}'>{{ game_link }}</a></div>
      </div>

    </body>

游戏的js客户端使用 `sae.Channel` 来创建一条channel连接，并且设置channel的onopen/onmessage/onerror/onclose的callback函数。

**使用channel来推送游戏状态信息**

当用户B点击用户A发过来的连接打开了游戏页面时，游戏的javascript客户端通过 `sendMessage` 函数通知服务端。

    onOpened = function() {
      connected = true;
      sendMessage('opened');
      updateBoard();
    };

    sendMessage = function(path, opt_param) {
      path += '?g=' + state.game_key;
      if (opt_param) {
        path += '&' + opt_param;
      }
      var xhr = new XMLHttpRequest();
      xhr.open('POST', path, true);
      xhr.send();
    };

服务端更新当前游戏的状态，并且通过channel的`send_message`将游戏的新的状态发送给用户A和用户B的channel客户端。客户端接受到消息后更新游戏页面。此后用户A和用户B交替走棋，客户端通过`sendMessage`将用户的走法发送给服务端。

    moveInSquare = function(id) {
      if (isMyMove() && state.board[id] == ' ') {
        sendMessage('/move', 'i=' + id);
      }
    }

服务收到消息后更新游戏的状态，再通过`send_message`将更新后的状态发送给用户A和B，如此往复直到游戏结束为止。

    class MovePage(tornado.web.RequestHandler):

      def post(self):
        game_key = self.get_argument('g')
        game = Game.get_by_key_name(game_key)
        user = self.get_secure_cookie('u')
        if game and user:
          id = int(self.get_argument('i'))
          GameUpdater(game).make_move(id, user)

    class GameUpdater():
      game = None

      def __init__(self, game):
        self.game = game

      def get_game_message(self):
        gameUpdate = {
          'board': self.game.board,
          'userX': self.game.userX,
          'userO': '' if not self.game.userO else self.game.userO,
          'moveX': self.game.moveX,
          'winner': self.game.winner,
          'winningBoard': self.game.winning_board
        }
        return json.dumps(gameUpdate)

      def send_update(self):
        message = self.get_game_message()
        channel.send_message(self.game.userX + self.game.key_name, message)
        if self.game.userO:
          channel.send_message(self.game.userO + self.game.key_name, message)

      def check_win(self):
        if self.game.moveX:
          # O just moved, check for O wins
          wins = Wins().o_wins
          potential_winner = self.game.userO
        else:
          # X just moved, check for X wins
          wins = Wins().x_wins
          potential_winner = self.game.userX
          
        for win in wins:
          if win.match(self.game.board):
            self.game.winner = potential_winner
            self.game.winning_board = win.pattern
            return

      def make_move(self, position, user):
        if position >= 0 and user == self.game.userX or user == self.game.userO:
          if self.game.moveX == (user == self.game.userX):
            boardList = list(self.game.board)
            if (boardList[position] == ' '):
              boardList[position] = 'X' if self.game.moveX else 'O'
              self.game.board = "".join(boardList)
              self.game.moveX = not self.game.moveX
              self.check_win()
              self.game.put()
              self.send_update()
              return

GameUpdater类检查move的请求是否合法，如果合法则更新游戏的状态并且通知游戏双方新的游戏状态。

注意事项：

1. 每个html页面最多可以建立1个channel连接。
2. 每个创建的channel只允许一个channel客户端连接。