exports.chi = function(userId,type){
    var seatData = gameSeatsOfUsers[userId];
    if(seatData == null){
        console.log("can't find user game data.");
        return;
    }

    var game = seatData.game;
    var chupai = game.chuPai;

    //如果当前游戏轮子是他自己，则忽略。这个？？？
    if(game.turn == seatData.seatIndex){
        console.log("it's your turn.");
        return;
    }

    //如果没有吃的机会，则不能吃
    if(seatData.canChi == false){
        console.log("seatData.canChi 是 false");
        return;
    }

    //胡了的，就不要再来了(这个可以在源头杜绝，即检查下家是否可吃的时候，判断下家是否是胡了的)
    if(seatData.hued){
        console.log('you have already hued. no kidding plz.');
        return;
    }
    
    //如果有人可以胡牌，则需要等待。
    //如果有人可以碰杠，则需要等待，因为当一张牌同时可以被吃和碰（杠）的时候，碰（杠）优先
    var i = game.turn;
    while(true){
        var i = (i + 1)%4;
        if(i == game.turn){
            break;
        }
        else{
            var ddd = game.gameSeats[i];
            if((ddd.canHu || ddd.canPeng || ddd.canGang) && i != seatData.seatIndex){
                return;    
            }
        }
    }

    clearAllOptions(game);

    //验证手上的牌的数目，并进行吃牌处理，扣掉手上的牌。
    var pais = seatData.canChi[type];
    seatData.eats = seatData.eats.concat(pais);

    pais.splice(pais.indexOf(game.chuPai),1);//canChi里面可能掺杂有此人并没有的牌，如上家打出的那张牌。
    for(var pai = 0; pai<pais.length; pai++){
        var c = seatData.countMap[pai];
        if(c == null || c == 0){
            console.log("lack of pai:",pai);
            return;
        }
        var index = seatData.holds.indexOf(pai);
        if(index == -1){
            console.log("can't find mj.");
            return;
        }
        seatData.holds.splice(index,1);
        seatData.countMap[pai] --;
    }

    game.chuPai = -1;

    recordGameAction(game,seatData.seatIndex,ACTION_CHI,chupai);

    //广播通知其它玩家
    userMgr.broacastInRoom('chi_notify_push',{userid:seatData.userId,pai:chupai},seatData.userId,true);

    //吃的玩家打牌
    moveToNextUser(game,seatData.seatIndex);
    
    //广播通知出牌方
    seatData.canChuPai = true;
    userMgr.broacastInRoom('game_chupai_push',seatData.userId,seatData.userId,true);
};
