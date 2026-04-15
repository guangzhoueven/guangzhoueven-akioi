function GameManager(size, InputManager, Actuator) {
  this.generateId = () => {
    let str = ''
    let charset = '0123456789qwertyuiopasdfghjklzxcvbnm'
    for(let i = 0; i < 16; i++) {
      str += charset[Math.floor(Math.random() * charset.length)]
    }
    return str
  }

  this.size         = size; // Size of the grid
  this.inputManager = new InputManager;
  this.actuator     = new Actuator;

  this.running      = false; // Auto step running
  this.cheated      = false; // Cheat enabled
  this.gameid       = this.generateId(); // Game ID, for ranklist stat
  this.steps        = 0; // Game steps
  this.history      = []; // History game state

  this.inputManager.on("move", this.move.bind(this));
  this.inputManager.on("restart", this.restart.bind(this));

  this.inputManager.on('think', function() {
    if(!this.cheated) {
      alert("请先开启作弊！");
      document.querySelector('#feedback-container').innerHTML='';
      return;
    }
    var best = this.ai.getBest();
    this.actuator.showHint(best.move);
  }.bind(this));
  
  this.inputManager.on('submit', () => {
    var username;
    var dd = new Date();
    var deltaT=dd.getTime() - localStorage['65536AI_const_username_date'];
    var prompted = false;
    if(
        !localStorage['65536AI_const_username'] || //未指定
        !localStorage['65536AI_const_username_date'] || //无日期
        deltaT > 30*86400*1000 //已经过期
    ) {
        username=prompt("【诚信提交，拒绝作弊】\n你的名字是？\n注意：名字设置后的 30 天内不可修改",localStorage['65536AI_username']);
        prompted = true;
    }
    else {
        username=localStorage['65536AI_const_username'];
        if(!confirm('【诚信提交，拒绝作弊】\n提交的用户名：'+username+'\n'+"您在 "+Math.floor(deltaT/86400/1000)+" 天前设置了这个名字")) username='';
    }
    if(!username) {return;}
    if(!username.match(/^(\w+)$/)) {alert("用户名只能包含字母、数字和下划线");return;}
    if(username.length < 2) {alert("用户名至少要有 2 的长度");return;}
    if(username.length > 14 && prompted) {alert("用户名长度不可超过 14");return;}
    if(this.cheated) {
      let cheatConfirm = false;
      if(localStorage['65536AI_submit_cheat']) {
        cheatConfirm = confirm('确定提交作弊的成绩？');
      } else {
        alert('不能提交通过启用作弊获得的成绩');
      }
      if(!cheatConfirm) return;
    }
    localStorage['65536AI_username'] = username;
    localStorage['65536AI_const_username'] = username;
    if(prompted) localStorage['65536AI_const_username_date'] = dd.getTime();
    var submitdata = JSON.stringify({
      gameid: this.gameid,
      finalGrid: JSON.stringify(this.grid.snapshot()),
      maxValueLog: this.grid.maxValue(),
      steps: this.steps,
      history: JSON.stringify(this.history),
      score: this.score,
      scoreStringForReference: '' + this.score,
      cheated: this.cheated
    });
    $.ajax({
      type: "POST",
      cache: false,
      url: "ranklist/submit_score.php",
      dataType: "json",
      data: {
        'username': username,
        'data': submitdata
      },
      error: function(e){
        console.log('提交成绩错误：',e);
        alert("出现错误，无法提交成绩。");
      },
      success: function(e){
        window.sendAnalyticsEvent('submit');
        console.log('提交成绩成功：',e);
        if(e.success) {
          alert("已经成功提交成绩，谢谢！\n你的排名为："+e.data+"，刷新即可看到您的成绩");
        }
        else {
          alert("提交失败，原因："+e.data);
        }
      }
    });
  });


  this.inputManager.on('run', function() {
    if(!this.cheated && !confirm('确定使用作弊功能？\n本次游戏的成绩将不计入最高分，并且不可提交到排行榜！')) return;
    if(!this.cheated) {this.cheated = true;this.actuator.setRunButton('手动');return;}
    if (this.running) {
      this.running = false;
      this.actuator.setRunButton('手动');
    } else {
      this.running = true;
      this.run()
      this.actuator.setRunButton('自动');
    }
  }.bind(this));

  this.setup(false);
}

// Restart the game
GameManager.prototype.restart = function () {
  this.actuator.restart();
  this.running = false;
  this.cheated = false;
  this.gameid = this.generateId();
  this.steps = 0;
  this.history = [];
  this.actuator.setRunButton('开启作弊');
  this.setup(true);
};

// Load game data
GameManager.prototype.loadGame = function (e) {
  if(!e.history) {
    return
  }
  console.info("Game loaded.");
  this.score=e.score;
  this.won=e.won;
  this.over=e.over;
  this.cheated=e.cheated;
  this.steps  = e.steps;
  this.history = e.history;
  if(e.gameid) this.gameid = e.gameid;
  if(this.cheated) {
    this.actuator.setRunButton('手动');
  }
  else {
    this.actuator.setRunButton('开启作弊');
  }
  for(var i=0;i<this.size;i++)
  {
    for(var j=0;j<this.size;j++)
    {
      if(e.grid.cells[i][j]==null) this.grid.cells[i][j]=null;
      else
      {
        this.grid.cells[i][j]=new Tile({x:i,y:j},e.grid.cells[i][j].value);
      }
    }
  }
}

// Set up the game
GameManager.prototype.setup = function (isrestart) {
  this.grid         = new Grid(this.size);
  let lastHistory = []
  this.history.push(lastHistory)
  lastHistory[0] = this.grid.snapshot()
  this.grid.addStartTiles();
  lastHistory[1] = this.grid.snapshot()

  this.ai           = new AI(this.grid);

  this.score        = 0;
  this.over         = false;
  this.won          = false;
  
  if(typeof(localStorage["65536AI_best"])=="undefined")
  {
    localStorage["65536AI_best"]="0";
  }
  if (
    typeof(localStorage["65536AI_gameSave"])!="undefined" &&
    localStorage["65536AI_gameSave"]!="null" &&
    !isrestart
  ){
    var e=JSON.parse(localStorage["65536AI_gameSave"]);
    this.loadGame(e);
  }

  // Update the actuator
  this.actuate();
};


// Sends the updated grid to the actuator
GameManager.prototype.actuate = function () {
  if(!this.cheated && this.score>parseInt(localStorage["65536AI_best"]))
  {
    localStorage["65536AI_best"]=this.score;
  }
  localStorage["65536AI_gameSave"]=JSON.stringify(this);
  this.actuator.actuate(this.grid, {
    score: this.score,
    over:  this.over,
    won:   this.won,
    best:  parseInt(localStorage["65536AI_best"])
  });
};

// makes a given move and updates state
GameManager.prototype.move = function(direction) {
  //console.info("Move: ");
  //console.info(this);
  var result = this.grid.move(direction);
  this.score += result.score;

  if(result.moved) {
    this.steps += 1;
  }

  let lastHistory = []
  if(result.moved) {
    this.history.push(lastHistory);
  }
  lastHistory[0] = this.grid.snapshot()
  if (result.moved) {
    if(result.won) {
      this.won = true
    }
    this.grid.computerMove();
  }
  lastHistory[1] = this.grid.snapshot()

  if (!this.won && !this.grid.movesAvailable()) {
    this.over = true; // Game over!
  }

  this.actuate();
}

// moves continuously until game is over
GameManager.prototype.run = function() {
  var best = this.ai.getBest();
  this.move(best.move);
  var timeout = animationDelay;
  if (this.running && !this.over && !this.won) {
    var self = this;
    setTimeout(function(){
      self.run();
    }, timeout);
  }
}
