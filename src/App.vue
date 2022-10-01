        
  <template>
  <div id="app" v-bind:class="clStatus">
  <h4>TPC iSeg Monitor (for control see vnc://ata03:1) </h4> 
                        Time: {{timestamp}}
                        <br/>

                        <input type="checkbox" id="canPlay" v-model="$data.canPlay"/>allow beep sound

  <table align=center>
  <tbody>

  <tr v-for="(item,key) in $data.value" :key=key>
  <td>{{key}}</td>
  <td width=50>{{item.Status_runningState_value}}</td>
  <td width=100 align=right>{{item.Status_voltageMeasure_value}}</td>
  <td width=10 align=right>{{item.Status_voltageMeasure_unit}}</td>
  <td width=100 align=right>{{item.Control_voltageSet_value}}</td>
  <td width=10 align=right>{{item.Control_voltageSet_unit}}</td>
  <td width=100 align=right>{{item.Status_currentMeasure_value}}</td>
  <td width=10 align=right>{{item.Status_currentMeasure_unit}}</td>
  <td width=100 align=right>{{item.Control_currentSet_value}}</td>
  <td width=10 align=right>{{item.Control_currentSet_unit}}</td>
  <td width=10 align=right>{{item.Event_currentTrip_value}}</td>
  <td width=10 align=right>{{item.Setup_delayedTripAction_value}}</td>
  </tr>
  </tbody>
  </table>
  </div>
</template>

<script>
export default {
    // (set-
  name: 'App',
  data: function() {
    return {
      isAllOK : true,
      clStatus : "normal",
      timestamp: "timestamp",
      connection: null,
      accessToken: null,
      time: "a",
      value: {},
      info: [],
      requestItems: [
//        "Status.on",
        "Status.ramping",
        "Status.voltageMeasure",
        "Status.currentMeasure",
        "Status.runningState",
        "Control.currentSet",
        "Control.voltageSet",
        "Setup.delayedTripAction",
        "Setup.delayedTripTime",
        "Event.currentTrip"
      ],
      beep: new Audio(),
      canPlay: false,
      
    }
  },
  methods: {
    createGetItemPacket : function (item,line,addr,channel) {
      var packet = {
        'c' : "getItem",
        'p' : {
          'p' : { 'l' : line, 'a': addr, 'c' : channel },
          'i' : item,
          'v' : "",
          'u' : ""
        }
      };
      return packet;
    },
    
    createEnvelope: function (id, type, content) {
      var envelope = { 'i' : id, 't' : type, 'c' : content, 'r' : "websocket" };
      return JSON.stringify(envelope,null,'\t');
    },
      
    sendMessage: function(message) {
      message;
        //        if (response.t == "response") {
        //          for (var i = 0; i < response.c.length; ++i) {
        //            var item = {};
        //            item.value = response.c[i].d.v;
        //            item.unit =  response.c[i].d.u;
        //            item.time =  response.c[i].d.t;
        //            item.line =  response.c[i].d.p.l;
        //            item.addr =  response.c[i].d.p.a;
        //            item.channel =  response.c[i].d.p.c;
        //            self.value[i] = item;
        //          }
        //          console.log(response.c);
    }
  },
  created: function() {
    console.log("Starting connection to WebSocket Server");
    this.connection = new WebSocket("ws://10.32.8.88:8080");
    
//    this.connection.onmessage = function(event) {
//      console.log("message sent");
//      console.log(event);
//    }
    const self = this;
    this.connection.onopen = function(event) {
      console.log(event);
      console.log("Successfully connected to the echo websocket server...");
      var envelope = {} ;
      envelope.i = "";
      envelope.t = "login";
      envelope.c = {
	"l": "admin",
	"p": "password",
	"t": ""
      };
      envelope.r = "websocket"; // choose websocket for push mode session or xhr for polling session
      var message = JSON.stringify(envelope,null,'\t');
      self.connection.send(message);
//      console.log(message);
      this.nextIsResponse = 2;
      self.connection.onmessage = function (evt) {
        console.log("MESSAGE: " + evt.data);
        var response = JSON.parse(evt.data);
        if (typeof (response.i) !== 'undefined') {
          // assign session ID to responding input field
          console.log(response.i);
          self.accessToken = response.i;
        }
        var content = [];
        self.requestItems.forEach( item => {
          content.push(self.createGetItemPacket(item,"*","*","*"));
        });
        var id = self.accessToken;
        console.log(id);
        var envelope = self.createEnvelope(id,"request",content);
        console.log(envelope);
        self.connection.send(envelope);
        self.connection.onmessage = function (evt) {
          var responses = JSON.parse(evt.data);
          self.time = responses;
          self.info = [];
          responses.forEach ( response => {
            if (response.t == "info" || response.t == "response") {
              response.c.forEach (content => {
                self.info.push(JSON.stringify(content,null,'\t'));
//                console.log(content.d.i);
//                console.log(self.requestItems.indexOf(content.d.i));
                if (self.requestItems.indexOf(content.d.i) != -1 && content.d.p.c != "") {
                  var ids = content.d.p;
                  var key = ids.l + "_" + ids.a + "_" + ids.c;
                  let ctype = content.d.i.replace(/\./,'_');
                  if (!(key in self.value)) {
                    self.value[key] = {};
                  }
//                  console.log(ctype);
//                  console.log(content.d);
//                  self.value[key]["Status_voltageMeasure"] = {};
                  //                  self.value[key]["Status_voltageMeasure"] = {'value': content.d.v, "unit": content.d.u, "time": content.d.t};
                  self.value[key][ctype + "_value"] = content.d.v;
                  self.value[key][ctype + "_unit"] = content.d.u;
                  self.value[key][ctype + "_time"] = content.d.t;
                    //{'value': content.d.v, "unit": content.d.u, "time": content.d.t};
                  self.value = JSON.parse(JSON.stringify(self.value));
//                  console.log("hoge:" + ctype + ":"+JSON.stringify(self.value[key][ctype]));
                }
                
              });
            }
          });        }
        ////        var responseText = JSON.stringify(response,null,'\t');
        //        console.log(this.nextIsResponse);
        //        if (response.t == 'response' || this.nextIsResponse > 0) {
        //          // assign session ID to responding input field
        ////          $('#response').html(responseText);
        //          this.nextIsResponse--;
        //        }
      }
    }
  },
  mounted: function () {
    this.beep.src = 'http://ata03/metis/media/beep.mp3';
    this.beep.load();

    const self = this;
    setInterval( function () {
      var now = new Date();
      self.timestamp = now.getFullYear()
        + "/" + ("00" + (now.getMonth() + 1)).slice(-2)
        + "/" + ("00" + now.getDate()).slice(-2)
        + " " + ("00" + now.getHours()).slice(-2)
        + ":" + ("00" + now.getMinutes()).slice(-2)
        + ":" + ("00" + now.getSeconds()).slice(-2);
    }, 1000);
    setInterval( function () {
      self.isAllOK = true;
      for (let key in self.value) {
        if ("Event_currentTrip_value" in self.value[key] && self.value[key].Event_currentTrip_value != 0) {
          self.isAllOK = false;
        }
        if (Math.abs(self.value["0_0_0"]["Status_voltageMeasure_value"]) < 7700) {
          self.isAllOK = false;
        }
      }
      if (self.isAllOK) {
        self.clStatus = "";
      } else {
          if (self.clStatus == "normal") {
            if (self.canPlay &&  (self.beep.ended || self.beep.currentTime == 0)) {
              self.beep.currentTime = 0;
              self.beep.play();
            }
            self.clStatus = "warning";
          } else {
            self.clStatus = "normal";
          }
      }
        
    },1000);
//    var content = [];
//    this.requestItems.forEach( item => {
//      content.push(this.createGetItemPacket(item,"*","*","*"));
//    });
//    var id = this.accessToken;
//    console.log(id);
//    var envelope = this.createEnvelope(id,"request",content);
//    console.log(envelope);
//    this.connection.send(envelope);
//    this.connection.onmessage = function (evt) {
//      var responses = JSON.parse(evt.data);
//      this.time = responses;
//      responses.forEach ( response => {
//        
//        console.log("hoge:" + response.t);
//      });
//      //        if (response.t == "response") {
      //          for (var i = 0; i < response.c.length; ++i) {
      //            var item = {};
      //            item.value = response.c[i].d.v;
      //            item.unit =  response.c[i].d.u;
      //            item.time =  response.c[i].d.t;
      //            item.line =  response.c[i].d.p.l;
      //            item.addr =  response.c[i].d.p.a;
      //            item.channel =  response.c[i].d.p.c;
      //            self.value[i] = item;
      //          }
      //          console.log(response.c);
//    }
    
  }
  
}
</script>

<style>
#app {
  font-family: 'MS Gothic';
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
  margin-top: 60px;
}

  .warning {
    background-color: #aa0000;
    color: #000000;
    text-align: center
  }
  .normal {
    background-color: #8888bb;
    color: #000000;
    text-align: center;
  }

</style>
