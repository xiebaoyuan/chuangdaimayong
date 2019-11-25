
var config = require('./config_reader.js').get_config().redis;
var redis  = require('redis');

var hosts = config.hosts;
var port = config.port;
var len = config.hosts.length;
var db = config.db;
var auth_pass = config.auth_pass;


var g_client = null;



function get_client(callback) {
	if (!g_client) {
        var opt = {no_ready_check : true};
        if (auth_pass){
            opt['password'] = auth_pass;
            opt['auth_pass'] = auth_pass;
        }
        if (db) {
            var dbSel = parseInt(db, 10);
            if (!isNaN(dbSel) || db < 0 || db > 15) {
                console.log('Redis init failed, db value invalid');
                callback('redis init failed');
                return;
            }
            opt['db'] = dbSel;
        }
        g_client = redis.createClient(port, hosts[parseInt(Math.random()*(len-1), 10)], opt);

		g_client.on("error", function (err) {
            g_client = null;
			console.log("Redis Error " + err.message);
		});

		callback(null, {client : g_client});
	} else {
		callback(null, {client : g_client});
	}

}

module.exports.redis_get_obj_value_nocheck = function(key,callback){
	get_client(function(err, info){

		if (err) {
			callback(err);
			return;
		};

		var client = info.client;

		if(!client){
			logger.warn('[redis_get_obj_value] get_client fail');
			return callback('Get redis client failed', null);
		}
		client.hgetall(key, function(err, value){
			if(err){
				logger.warn('[redis_get_obj_value]hgetall redis failed! [err: %s]', err);
				return callback('Get redis failed', null);
			}
			callback(null,value);
		});

	});
}

module.exports.redis_get_obj_value = function(key,callback){
	get_client(function(err, info){

		if (err) {
			callback(err);
			return;
		};

		var client = info.client;
		if(!client){
			logger.warn('[redis_get_obj_value] get_client fail');
			return callback('Get redis client failed', null);
		}
		client.hgetall(key, function(err, value){
			if(err){
				logger.warn('[redis_get_obj_value]hgetall redis failed! [err: %s]', err);
				return callback('Get redis failed', null);
			}
			if(value){
				callback(null,value);
			}else{
				callback(null,null);
			}
		});
	});
}


module.exports.redis_set_obj_value = function(key, p_value, live_time, callback) {
	get_client(function(err, info){
		if (err) {
			callback(err);
			return;
		};

		var client = info.client;
		if(!client){
			logger.warn('[redis_set_obj_value] get_client fail');
			return callback('Get redis client failed', null);
		}
		client.hmset(key, p_value,function(err, value){
			if(err){
				logger.warn('[redis_set_obj_value]hmset redis failed! [err: %s]', err);
				return callback('Set redis failed', null);
			}
			if(live_time){
				client.expire(key, live_time);
			}
			callback(null,null);
		});
	});
}

module.exports.redis_del_all_value = function(key,callback){
	get_client(function(err, info){

		if (err) {
			callback(err);
			return;
		};

		var client = info.client;
		if(!client){
			logger.warn('[redis_del_all_value] get_client fail');
			return callback('Get redis client failed');
		}
		client.del(key,function(err, value){
			if(err){
				logger.warn('[redis_set_obj_value]hmset redis failed! [err: %s]', err);
				return callback('Set redis failed');
			}
			callback(null);
		});
	});
}


module.exports.redis_get_value = function(key,callback){
	get_client(function(err, info){

		if (err) {
			callback(err);
			return;
		};

		var client = info.client;
		if(!client){
			logger.warn('[redis_get_value] get_client fail');
			return callback('Get redis client failed', null);
		}
		client.get(key, function(err, value){
			if(err){
				logger.warn('[redis_get_value]get redis failed! [err: %s]', err);
				return callback('Get redis failed', null);
			}
			if(value){
				callback(null,value);
			}else{
				callback(null,null);
			}
		});
	});
}

module.exports.redis_set_value = function(key, p_value, live_time, callback) {
	get_client(function(err, info){

		if (err) {
			callback(err);
			return;
		};

		var client = info.client;
		if(!client){
			logger.warn('[redis_set_value] get_client fail');
			return callback('Get redis client failed', null);
		}
		client.set(key, p_value,function(err, value){
			if(err){
				logger.warn('[redis_set_value] set redis failed! [err: %s]', err);
				return callback('Set redis failed', null);
			}
			if(live_time){
				client.expire(key, live_time);
			}
			callback(null,null);
		});
	});
}

module.exports.keys = function(pattern, callback){
    get_client(function(err, info){

        if (err) {
            callback(err);
            return;
        };

        var client = info.client;
        if(!client){
            logger.warn('[keys] get_client fail');
            return callback('Get redis client failed', null);
        }
        client.keys(pattern, function(err, value){
            if(err){
                logger.warn('[keys]get redis failed! [err: %s]', err);
                return callback('Get redis failed', null);
            }
            
            callback(null, value);
        });
    });
}

module.exports.rpush = function(key, value, callback){
    get_client(function(err, info){

        if (err) {
            callback(err);
            return;
        };

        var client = info.client;
        if(!client){
            logger.warn('[rpush] get_client fail');
            return callback('Get redis client failed', null);
        }
        client.rpush(key, value, function(err, r){
            if(err){
                logger.warn('[rpush]get redis failed! [err: %s]', err);
                return callback('Get redis failed', null);
            }
            
            callback(null, r);
        });
    });
}

module.exports.lrem = function(key, cnt, value, callback){
    get_client(function(err, info){

        if (err) {
            callback(err);
            return;
        };

        var client = info.client;
        if(!client){
            logger.warn('[lrem] get_client fail');
            return callback('Get redis client failed', null);
        }
        client.lrem(key, cnt, value, function(err, r){
            if(err){
                logger.warn('[lrem]get redis failed! [err: %s]', err);
                return callback('Get redis failed', null);
            }
            
            callback(null, r);
        });
    });
}

module.exports.lrem = function(key, cnt, value, callback){
    get_client(function(err, info){

        if (err) {
            callback(err);
            return;
        };

        var client = info.client;
        if(!client){
            logger.warn('[lrem] get_client fail');
            return callback('Get redis client failed', null);
        }
        client.lrem(key, cnt, value, function(err, r){
            if(err){
                logger.warn('[lrem]get redis failed! [err: %s]', err);
                return callback('Get redis failed', null);
            }
            
            callback(null, r);
        });
    });
}

module.exports.lrange = function(key, start, end, callback){
    get_client(function(err, info){

        if (err) {
            callback(err);
            return;
        };

        var client = info.client;
        if(!client){
            logger.warn('[lrange] get_client fail');
            return callback('Get redis client failed', null);
        }
        client.lrange(key, start, end, function(err, r){
            if(err){
                logger.warn('[lrange]get redis failed! [err: %s]', err);
                return callback('Get redis failed', null);
            }
            
            callback(null, r);
        });
    });
}

module.exports.llen = function(key, callback){
    get_client(function(err, info){

        if (err) {
            callback(err);
            return;
        };

        var client = info.client;
        if(!client){
            logger.warn('[llen] get_client fail');
            return callback('Get redis client failed', null);
        }
        client.llen(key, function(err, r){
            if(err){
                logger.warn('[llen]get redis failed! [err: %s]', err);
                return callback('Get redis failed', null);
            }
            
            callback(null, r);
        });
    });
}
/**
* 用于将数据放入有序set中
* 基础结构为
* structure : {key : xxxx, score : iii}
* data的结构和structure相同或由structure构成的数组
*/

module.exports.zadd = function(set_name, data, callback){

    if (!set_name || !data) {
        callback('set name undefined');
        return;
    };

    var args = [set_name];
    if (data instanceof Array) {
        for (var i = 0; i < data.length; i++) {
            if (undefined == data[i].score || !data[i].key) {
                callback('data structure error');
                return;
            };
            args.push(data[i].score);
            args.push(data[i].key);
        };
    } else {
        if (undefined == data.score || !data.key) {
            callback('data structure error');
            return;
        };
        args.push(data.score);
        args.push(data.key);
    }

    if (args.length == 1) {
        console.log('redis zadd args length == 1');
        return callback(null);
    };

    get_client(function(err, info){

        if (err) {
            callback(err);
            return;
        };

        var client = info.client;
        if(!client){
            console.log('[zadd] get_client fail');
            return callback('Get redis client failed', null);
        }

        client.zadd(args, function(err, response){

            if (err) {
                console.dir(err);
                callback('redis zadd operation failed');
                return;
            };

            callback(null, null);

        });

    });

}

/**
* 提供min和max，从有序set中取得范围内的key值
* 输入参数restrictions，表示要获取数据的范围，结构如下：
* {min : xxx, max : xxx}（包括min和max）
*/
module.exports.zrangebyscore = function(set_name, restrictions, cb){
    var min = '-inf';
    var max = '+inf';
    var limit;
    var offset = 0;
    var callback = cb;

    if (!set_name) {
        callback && callback('param error, set_name invalid');
        return;
    };

    if (typeof restrictions == 'function') {
        callback = restrictions;
    } else {
        if (undefined != restrictions.min) {
            min = restrictions.min;
        };

        if (undefined != restrictions.max) {
            max = restrictions.max;
        };

        if (undefined != restrictions.limit) {
            limit = restrictions.limit;
            offset = restrictions.offset || 0;
        };
    }

    var args = [set_name, min, max];

    if (limit) {
        args.push('LIMIT');
        args.push(offset);
        args.push(limit);
    };
    
    get_client(function(err, info){

        if (err) {
            callback && callback(err);
            return;
        };

        var client = info.client;
        if(!client){
            console.log('[zrangebyscore] get_client fail');
            return callback && callback('Get redis client failed', null);
        }

        client.zrangebyscore(args, function(err, response){

            if (err) {
                console.dir(err);
                callback && callback('redis zrangebyscore operation failed');
                return;
            };

            callback && callback(null, response);

        });

    });

}

/**
* 提供一个max和min，从有序set中删除范围内的序列
* （包括min和max）
*/
module.exports.zremrangebyscore = function(set_name, restrictions, cb){
    var min = '-inf';
    var max = '+inf';
    var callback = cb;

    if (!set_name) {
        callback && callback('param error, set_name invalid');
        return;
    };

    if (typeof restrictions == 'function') {
        callback = restrictions;
    } else {
        if (undefined != restrictions.min) {
            min = restrictions.min;
        };

        if (undefined != restrictions.max) {
            max = restrictions.max;
        };
    }

    var args = [set_name, min, max];
    
    get_client(function(err, info){

        if (err) {
            callback && callback(err);
            return;
        };

        var client = info.client;
        if(!client){
            console.log('[zremrangebyscore] get_client fail');
            return callback && callback('Get redis client failed', null);
        }

        client.zremrangebyscore(args, function(err, response){

            if (err) {
                console.dir(err);
                callback && callback('redis zremrangebyscore operation failed');
                return;
            };

            callback && callback(null, null);

        });

    });

}

/**
* 传入要删除的建值
*/
module.exports.zrem = function(set_name, key, callback){
    if (!set_name || !key) {
        console.log('[zrem] param invalid');
        callback('param invalid');
        return;
    };

    var args = [set_name, key];
    
    get_client(function(err, info){

        if (err) {
            callback && callback(err);
            return;
        };

        var client = info.client;
        if(!client){
            console.log('[zrem] get_client fail');
            callback && callback('Get redis client failed', null);
            return;
        }

        client.zrem(args, function(err, response){

            if (err) {
                console.dir(err);
                callback && callback('redis zrem operation failed');
                return;
            };

            callback && callback(null, null);

        });

    });

}

/**
* mget
* 传入多个key值，批量获取对应的value，
* 输入参数 字符串或key数组
*/
module.exports.mget = function(keys, callback){

	if (!keys) {
        console.log('[mget] param invalid');
        callback && callback('param invalid');
        return;
    };

    var args = [];
    if (keys instanceof Array) {
    	args = keys;
    } else if ('string' == typeof(keys)){
    	args.push(keys);
    } else {
    	console.log('param type invalid');
    	callback && callback('param type invalid');
    	return;
    }
    
    get_client(function(err, info){

        if (err) {
            callback && callback(err);
            return;
        };

        var client = info.client;
        if(!client){
            console.log('[mget] get_client fail');
            callback && callback('Get redis client failed', null);
            return;
        }

        client.mget(args, function(err, response){

            if (err) {
                console.dir(err);
                callback && callback('redis mget operation failed');
                return;
            };

            callback && callback(null, response);

        });

    });

}


/**
* mset
* 传入多个k-v，批量设置（注意批量设置的最大量最好不要超过5000）
* 输入参数：k-v数组
*/
module.exports.mset = function(args, callback){
	
	if (!args) {
        console.log('[mset] param invalid');
        callback && callback('param invalid');
        return;
    };

    if (args instanceof Array) {

    } else {
    	console.log('param type invalid');
    	callback && callback('param type invalid');
    	return;
    }
    
    get_client(function(err, info){

        if (err) {
            callback && callback(err);
            return;
        };

        var client = info.client;
        if(!client){
            console.log('[mset] get_client fail');
            callback && callback('Get redis client failed', null);
            return;
        }

        client.mset(args, function(err, response){

            if (err) {
                console.dir(err);
                callback && callback('redis mset operation failed');
                return;
            };

            callback && callback(null, null);

        });

    });

}

module.exports.incr = function(key, callback){
    if (!key || typeof key != 'string') {
        console.log('[incr] param invalid');
        callback && callback('param invalid');
        return;
    };
    
    get_client(function(err, info){

        if (err) {
            callback && callback(err);
            return;
        };

        var client = info.client;
        if(!client){
            console.log('[incr] get_client fail');
            callback && callback('Get redis client failed', null);
            return;
        }

        client.incr(key, function(err, response){

            if (err) {
                console.dir(err);
                callback && callback('redis incr operation failed');
                return;
            };

            callback && callback(null, response);

        });

    });
}

/**
* getset
* key：redis的k
* p_value: redis的v
* 返回值：key设置之前的值
*/
module.exports.getset = function(key, p_value, callback) {

    get_client(function(err, info){

        if (err) {
            callback && callback(err);
            return;
        };

        var client = info.client;
        if(!client){
            logger.warn('[getset] get_client fail');
            callback && callback('Get redis client failed', null);
            return;
        }
        client.getset(key, p_value,function(err, value){
            if(err){
                logger.warn('[getset] set redis failed! [err: %s]', err);
                callback && callback('Set redis failed', null);
                return;
            }

            //callback && callback(null,value);
            callback && callback(null,'为方便测试而临时添加，日后删除');
        });
    });

}
