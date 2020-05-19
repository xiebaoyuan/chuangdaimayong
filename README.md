package main

import (
	//"bytes"
	//"encoding/binary"
	//"encoding/json"
	"fmt"
	"github.com/go-redis/redis"
	"strconv"
	"strings"
	"sync"
	"time"
)

// 命令类型
type RedisCmd int
type RedisContainerType int

type redisPipelineFn func(redis.Pipeliner) error


// 命令字
const (
	REDIS_GET RedisCmd = iota
	REDIS_SET
	REDIS_SETNX
	REDIS_DEL
	REDIS_HGETALL // 得到完整的map数据
	REDIS_HGET    //得到hash对象中的一个field
	REDIS_HMGET
	REDIS_HSET
	REDIS_HMSET
	REDIS_HDEL // 删除hash对象中的一个或多个field
	REDIS_LGET // 得到完整的list数据
	REDIS_LSET // list的整体更新，把原有的key删除，重新设置
	REDIS_LRPUSH
	REDIS_LLPUSH
	REDIS_LREM      // 移除列表元素
	REDIS_LTRIM     // 保留列表指定区间的元素
	REDIS_LLEN      // 获取列表长度
	REDIS_LPOP      // 移出并获取第一个元素
	REDIS_SGET      // set的整体获取
	REDIS_SSET      // set的整体更新，把原有的key删除，重新设置
	REDIS_SISMEMBER // 判断是否再set中
	REDIS_SADD      // set add
	REDIS_SCARD
	REDIS_SMEMBERS
	REDIS_SREM // set remove
	REDIS_PIPELINE
	REDIS_CLUSTERSLOTS
	REDIS_EXISTS // 判断key是否存在
	REDIS_SCRIPT_LOAD
	REDIS_EVAlSHA
	REDIS_INCR
	REDIS_INCRBY
	REDIS_ZADD
	REDIS_ZINCRBY         // 有序集合中对指定成员的分数加上增量 increment
	REDIS_ZSCORE          // 返回有序集中，成员的分数值
	REDIS_ZREM            // 移除有序集中的一个或多个成员
	REDIS_ZREMRANGEBYRANK //根据score移除有序集合中的成员
	REDIS_EXPIRE          // 设置Key生存时间
	REDIS_ZRRANK          // 返回倒序索引
	REDIS_ZRANGEBYSCORE   //根据分数返回指定范围内的数据
	REDIS_SCAN            // key扫描
	REDIS_CLUSTERSCAN     // cluster scan
	REDIS_DBSIZE          // key的数量
	REDIS_PUBLISH
)

const redisSvrAddrs string = "192.168.0.201:6379,192.168.0.201:6380,192.168.0.201:6381"

func ErrTimeOut(key string) error {
	return fmt.Errorf("key[%s] time1 out!", key)
}

// 内部返回类型
type RedisResultS struct {
	Ok     bool
	Result interface{}
}

type RedisCommandData struct {
	cmd       RedisCmd
	key       string
	fieldKey  string
	value     interface{}
	ttl       int
	args      []interface{}
	replyChan chan<- *RedisResultS
}

func InitRedis(redisMgrr *RedisMgr,redisMgrLock *RedisMgr) bool {

	redisMgrrr, err := createRedisMgr()
	if err != nil {
		fmt.Println( "failed createRedisMgr")
		return false
	}

	//redisMgrr = redisMgrrr //①那里联动赋值不成功
	*redisMgrr = *redisMgrrr //只有这样，①那里才能联动赋值成功

	fmt.Println("create redismgr cluster sucessesd")

	redisMgrLockk, err := createRedisMgrLock()
	if err != nil {
		fmt.Println("failed createRedisMgrLock")
		return false
	}
	//logs.LogErrorFuncCallback = PublishErrorMsgToRedis
	*redisMgrLock = *redisMgrLockk

	return true
}

func createRedisMgr() (*RedisMgr, error) {

	fmt.Println( "redisSvrAddrs:", redisSvrAddrs)

	redisMgr := NewRedisMgr(strings.Split(redisSvrAddrs, ","), 500, true, "GEFpedXGCs2a4sN")
	err := redisMgr.Start()
	if err != nil {
		fmt.Println( "redisMgr[%s] start failed! reason:%v", redisSvrAddrs, err)
		return nil, err
	}
	fmt.Println( "Create NewRedisMgr successed!")
	return redisMgr, nil
}

func createRedisMgrLock() (*RedisMgr, error) {
	redisLockMgr := NewRedisMgr(strings.Split(redisSvrAddrs, ","), 500, false, "GEFpedXGCs2a4sN")
	err := redisLockMgr.Start()
	if err != nil {
		fmt.Println("redisMgr[%s] start failed! reason:%v", redisSvrAddrs, err)
		return nil, err
	}
	fmt.Println("Create redisLockMgr successed!")
	return redisLockMgr, nil
}

type RedisScanResult struct {
	keys   []string
	cursor uint64
}
const _RedisCmd_name = "REDIS_GETREDIS_SETREDIS_SETNXREDIS_DELREDIS_HGETALLREDIS_HMGETREDIS_HSETREDIS_HMSETREDIS_HDELREDIS_LGETREDIS_LSETREDIS_LRPUSHREDIS_LLPUSHREDIS_LREMREDIS_LTRIMREDIS_LLENREDIS_LPOPREDIS_SGETREDIS_SSETREDIS_SADDREDIS_SREMREDIS_PIPELINEREDIS_CLUSTERSLOTSREDIS_EXISTSREDIS_SCRIPT_LOADREDIS_EVAlSHAREDIS_INCRREDIS_ZINCRBYREDIS_ZSCOREREDIS_ZREMREDIS_EXPIREREDIS_ZRRANKREDIS_SCANREDIS_CLUSTERSCANREDIS_DBSIZE"

var _RedisCmd_index = [...]uint16{0, 9, 18, 29, 38, 51, 62, 72, 83, 93, 103, 113, 125, 137, 147, 158, 168, 178, 188, 198, 208, 218, 232, 250, 262, 279, 292, 302, 315, 327, 337, 349, 361, 371, 388, 400}

func (i RedisCmd) String() string {
	if i < 0 || i >= RedisCmd(len(_RedisCmd_index)-1) {
		return "RedisCmd(" + strconv.FormatInt(int64(i), 10) + ")"
	}
	return _RedisCmd_name[_RedisCmd_index[i]:_RedisCmd_index[i+1]]
}









type RedisMgr struct {
	redisSvrAddrs      []string
	isCluster          bool
	sessionCount       int
	password           string
	redisClient        *redis.Client
	redisClusterClient *redis.ClusterClient
	redisCmdable       redis.Cmdable
	redisCmdChan       chan *RedisCommandData
	redisExitChan      chan struct{}
	redisRoutineWG     sync.WaitGroup
}

func NewRedisMgr(redisSvrAddrs []string, sessionCount int, isCluster bool, password string) *RedisMgr {
	redisMgr := new(RedisMgr)
	redisMgr.redisSvrAddrs = redisSvrAddrs
	redisMgr.sessionCount = sessionCount
	redisMgr.isCluster = isCluster
	redisMgr.password = password
	redisMgr.redisCmdChan = make(chan *RedisCommandData, 1024)
	redisMgr.redisExitChan = make(chan struct{})
	return redisMgr
}
func (rm *RedisMgr) Start() error {
	fmt.Println("redisSvrAddrs:", rm.redisSvrAddrs)

	if rm.isCluster {
		rm.redisClusterClient = redis.NewClusterClient(&redis.ClusterOptions{
			Addrs: rm.redisSvrAddrs,
			OnConnect: func(conn *redis.Conn) error {
				fmt.Println("redisClusterClient redis.Conn:%s", conn.String())
				return nil
			},
			Password: rm.password,
			ReadOnly: false,
		})
		rm.redisCmdable = rm.redisClusterClient
	} else {
		rm.redisClient = redis.NewClient(&redis.Options{
			Addr:     rm.redisSvrAddrs[0],
			Password: rm.password,
			OnConnect: func(conn *redis.Conn) error {
				fmt.Println("redisClient redis.Conn:%s", conn.String())
				return nil
			},
		})
		rm.redisCmdable = rm.redisClient
	}

	var index = 0
	for index < rm.sessionCount {
		// 启动命令处理协程
		go redisCmdPerformRoutine(rm)
		rm.redisRoutineWG.Add(1)
		index++
	}

	var err error
	var res string
	if rm.isCluster {
		res, err = rm.redisClusterClient.Ping().Result()
		fmt.Println("redisClusterClient Ping:%s", res)
	} else {
		res, err = rm.redisClient.Ping().Result()
		fmt.Println("redisClient Ping:%s", res)
	}

	return err
}

func redisCmdPerformRoutine(rm *RedisMgr) {
	//fmt.Println("redisCmdPerformRoutine running")

	defer func() {
		rm.redisRoutineWG.Done()
		if err := recover(); err != nil {
			// 回收panic，防止整个服务panic

		}
	}()
L:
	for {
		select {
		case redisCmdData, ok := <-rm.redisCmdChan:

			if ok {
				switch redisCmdData.cmd {
				case REDIS_GET:
					val, err := rm.redisCmdable.Get(redisCmdData.key).Result()
					if err != nil {
						//fmt.Println("Get key[%s] failed! error:%s", redisCmdData.key, err.Error())
						redisCmdData.replyChan <- &RedisResultS{Ok: false, Result: err}
					} else {
						redisCmdData.replyChan <- &RedisResultS{Ok: true, Result: val}
					}
				case REDIS_SET:
					// 设置key value
					expiration := time.Duration(redisCmdData.ttl) * time.Second
					_, err := rm.redisCmdable.Set(redisCmdData.key, redisCmdData.value, expiration).Result()
					if err == nil {
						// set成功
						redisCmdData.replyChan <- &RedisResultS{Ok: true}
					} else {
						fmt.Println("Set key[%s] failed! error:%s", redisCmdData.key, err.Error())
						redisCmdData.replyChan <- &RedisResultS{Ok: false, Result: err}
					}
				case REDIS_DEL:
					res, err := rm.redisCmdable.Del(redisCmdData.key).Result()
					fmt.Println("Del key[%s] res[%d]", redisCmdData.key, res)
					if err != nil {
						redisCmdData.replyChan <- &RedisResultS{Ok: false, Result: err}
					} else {
						redisCmdData.replyChan <- &RedisResultS{Ok: true, Result: res}
					}
				case REDIS_SETNX:
					// 设置key value
					expiration := time.Duration(redisCmdData.ttl) * time.Second
					_, err := rm.redisCmdable.SetNX(redisCmdData.key, redisCmdData.value, expiration).Result()
					if err == nil {
						// set成功
						redisCmdData.replyChan <- &RedisResultS{Ok: true}
					} else {
						fmt.Println("SetNX key[%s] failed! error:%s", redisCmdData.key, err.Error())
						redisCmdData.replyChan <- &RedisResultS{Ok: false, Result: err}
					}
				case REDIS_HGETALL:
					val, err := rm.redisCmdable.HGetAll(redisCmdData.key).Result()
					if err == nil {
						// set成功
						redisCmdData.replyChan <- &RedisResultS{Ok: true, Result: val}
					} else {
						fmt.Println("HGETALL key[%s] failed! error:%s", redisCmdData.key, err.Error())
						redisCmdData.replyChan <- &RedisResultS{Ok: false, Result: err}
					}
				case REDIS_HGET:
					val, err := rm.redisCmdable.HGet(redisCmdData.key, redisCmdData.fieldKey).Result()
					if err == nil {
						redisCmdData.replyChan <- &RedisResultS{Ok: true, Result: val}
					} else {
						//fmt.Println(lc, "HGet key[%s] fieldKey[%s] failed! error:%s", redisCmdData.key, redisCmdData.fieldKey, err.Error())
						redisCmdData.replyChan <- &RedisResultS{Ok: false, Result: err}
					}
				case REDIS_HMGET:
				case REDIS_HMSET:
					res, err := rm.redisCmdable.HMSet(redisCmdData.key, redisCmdData.value.(map[string]interface{})).Result()
					//fmt.Println("HMSET key[%s] res:%v", redisCmdData.key, res)
					if err == nil {
						// set成功
						redisCmdData.replyChan <- &RedisResultS{Ok: true, Result: res}
					} else {
						fmt.Println("HMSET key[%s] failed! error:%s", redisCmdData.key, err.Error())
						redisCmdData.replyChan <- &RedisResultS{Ok: false, Result: err}
					}
				case REDIS_HSET:
					res, err := rm.redisCmdable.HSet(redisCmdData.key, redisCmdData.fieldKey, redisCmdData.value).Result()
					fmt.Println("HSET key[%s] fieldkey[%s] res:%v", redisCmdData.key, redisCmdData.fieldKey, res)
					if err == nil {
						// set成功
						redisCmdData.replyChan <- &RedisResultS{Ok: true, Result: res}
					} else {
						fmt.Println("HSET key[%s] fieldKey[%s] failed! error:%s", redisCmdData.key,
							redisCmdData.fieldKey, err.Error())
						redisCmdData.replyChan <- &RedisResultS{Ok: false, Result: err}
					}
				case REDIS_HDEL:
					res, err := rm.redisCmdable.HDel(redisCmdData.key, redisCmdData.value.([]string)...).Result()
					fmt.Println("HDEL key[%s] fieldkey or fieldkeys res:%v", redisCmdData.key, res)
					if err == nil {
						// HDEL成功
						redisCmdData.replyChan <- &RedisResultS{Ok: true, Result: res}
					} else {
						fmt.Println("HSET key[%s] fieldkey or fieldkeys failed! error:%s", redisCmdData.key,
							redisCmdData.fieldKey, err.Error())
						redisCmdData.replyChan <- &RedisResultS{Ok: false, Result: err}
					}
				case REDIS_EXISTS:
					val, err := rm.redisCmdable.Exists(redisCmdData.key).Result()
					if err != nil {
						fmt.Println("Exists cmds:%v", err)
						redisCmdData.replyChan <- &RedisResultS{Ok: false, Result: err}
					} else {
						redisCmdData.replyChan <- &RedisResultS{Ok: true, Result: val}
					}
				case REDIS_ZADD:
					fmt.Println("REDIS_ZADD key[%s] fieldKey[%s] score[%f]", redisCmdData.key, redisCmdData.fieldKey,
						redisCmdData.value.(float64))
					rz := redis.Z{
						Score:  redisCmdData.value.(float64),
						Member: redisCmdData.fieldKey,
					}
					val, err := rm.redisCmdable.ZAdd(redisCmdData.key, &rz).Result()
					if err != nil {
						redisCmdData.replyChan <- &RedisResultS{Ok: false, Result: err}
					} else {
						redisCmdData.replyChan <- &RedisResultS{Ok: true, Result: val}
					}
				case REDIS_ZREM:
					val, err := rm.redisCmdable.ZRem(redisCmdData.key, redisCmdData.value).Result()
					if err != nil {
						redisCmdData.replyChan <- &RedisResultS{Ok: false, Result: err}
					} else {
						redisCmdData.replyChan <- &RedisResultS{Ok: true, Result: val}
					}
				case REDIS_ZREMRANGEBYRANK:
					val, err := rm.redisCmdable.ZRemRangeByRank(redisCmdData.key,
						redisCmdData.args[0].(int64), redisCmdData.args[1].(int64)).Result()
					if err != nil {
						redisCmdData.replyChan <- &RedisResultS{Ok: false, Result: err}
					} else {
						redisCmdData.replyChan <- &RedisResultS{Ok: true, Result: val}
					}
				case REDIS_ZINCRBY:
					// 当 key 不存在，或 member 不是 key 的成员时，相当于zadd
					val, err := rm.redisCmdable.ZIncrBy(redisCmdData.key, redisCmdData.value.(float64), redisCmdData.fieldKey).Result()
					fmt.Println("ZIncrBy key[%s] increment[%v] memberKey[%s] val:%v", redisCmdData.key, redisCmdData.value,
						redisCmdData.fieldKey, val)
					if err != nil {
						redisCmdData.replyChan <- &RedisResultS{Ok: false, Result: err}
					} else {
						redisCmdData.replyChan <- &RedisResultS{Ok: true, Result: val}
					}
				case REDIS_ZRANGEBYSCORE:
					//fmt.Println("REDIS_ZRANGEBYSCORE key[%s]", redisCmdData.key)
					val, err := rm.redisCmdable.ZRangeByScore(redisCmdData.key, redisCmdData.value.(*redis.ZRangeBy)).Result()
					/*fmt.Println("REDIS_ZRANGEBYSCORE key[%s] val:%v", redisCmdData.key, redisCmdData.value,
					redisCmdData.fieldKey, val)*/
					if err != nil {
						redisCmdData.replyChan <- &RedisResultS{Ok: false, Result: err}
					} else {
						redisCmdData.replyChan <- &RedisResultS{Ok: true, Result: val}
					}
				case REDIS_EXPIRE:
					res, err := rm.redisCmdable.Expire(redisCmdData.key, redisCmdData.value.(time.Duration)).Result()
					//fmt.Println("Expire key[%s] expiration:%v", redisCmdData.key, redisCmdData.value)
					if err != nil {
						redisCmdData.replyChan <- &RedisResultS{Ok: false, Result: err}
					} else {
						redisCmdData.replyChan <- &RedisResultS{Ok: true, Result: res}
					}
				case REDIS_DBSIZE:
					res, err := rm.redisCmdable.DBSize().Result()
					fmt.Println("DBSize count:%d", res)
					if err != nil {
						redisCmdData.replyChan <- &RedisResultS{Ok: false, Result: err}
					} else {
						redisCmdData.replyChan <- &RedisResultS{Ok: true, Result: res}
					}
				case REDIS_SCAN:
					scanRes := new(RedisScanResult)
					var err error
					scanRes.keys, scanRes.cursor, err = rm.redisCmdable.Scan(redisCmdData.args[0].(uint64),
						redisCmdData.args[1].(string),
						redisCmdData.args[2].(int64)).Result()
					fmt.Println("Scan keycount[%d] cursor[%d]", len(scanRes.keys), scanRes.cursor)
					if err != nil {
						redisCmdData.replyChan <- &RedisResultS{Ok: false, Result: err}
					} else {
						redisCmdData.replyChan <- &RedisResultS{Ok: true, Result: scanRes}
					}
				case REDIS_CLUSTERSCAN:
					scanRes := new(RedisScanResult)
					var err error
					cmdable := redisCmdData.args[0].(redis.Cmdable)
					scanRes.keys, scanRes.cursor, err = cmdable.Scan(
						redisCmdData.args[1].(uint64),
						redisCmdData.args[2].(string),
						redisCmdData.args[3].(int64)).Result()
					fmt.Println("Scan keycount[%d] cursor[%d] err:%v", len(scanRes.keys), scanRes.cursor, err)
					if err != nil {
						redisCmdData.replyChan <- &RedisResultS{Ok: false, Result: err}
					} else {
						redisCmdData.replyChan <- &RedisResultS{Ok: true, Result: scanRes}
					}
				case REDIS_SADD:
					res, err := rm.redisCmdable.SAdd(redisCmdData.key, redisCmdData.args...).Result()
					fmt.Println("SADD key[%s] res:%d", redisCmdData.key, res)
					if err != nil {
						redisCmdData.replyChan <- &RedisResultS{Ok: false, Result: err}
					} else {
						redisCmdData.replyChan <- &RedisResultS{Ok: true, Result: res}
					}
				case REDIS_SREM:
					res, err := rm.redisCmdable.SRem(redisCmdData.key, redisCmdData.args...).Result()
					fmt.Println("SREM key[%s] res:%d", redisCmdData.key, res)
					if err != nil {
						redisCmdData.replyChan <- &RedisResultS{Ok: false, Result: err}
					} else {
						redisCmdData.replyChan <- &RedisResultS{Ok: true, Result: res}
					}
				case REDIS_SISMEMBER:
					res, err := rm.redisCmdable.SIsMember(redisCmdData.key, redisCmdData.value).Result()
					fmt.Println("SISMEMBER key[%s] res:%v", redisCmdData.key, res)
					if err != nil {
						redisCmdData.replyChan <- &RedisResultS{Ok: false, Result: err}
					} else {
						redisCmdData.replyChan <- &RedisResultS{Ok: true, Result: res}
					}
				case REDIS_SCARD:
					res, err := rm.redisCmdable.SCard(redisCmdData.key).Result()
					fmt.Println("SCard key[%s] res:%v", redisCmdData.key, res)
					if err != nil {
						redisCmdData.replyChan <- &RedisResultS{Ok: false, Result: err}
					} else {
						redisCmdData.replyChan <- &RedisResultS{Ok: true, Result: res}
					}
				case REDIS_SMEMBERS:
					res, err := rm.redisCmdable.SMembers(redisCmdData.key).Result()
					if err != nil {
						redisCmdData.replyChan <- &RedisResultS{Ok: false, Result: err}
					} else {
						redisCmdData.replyChan <- &RedisResultS{Ok: true, Result: res}
					}
				case REDIS_INCR:
					res, err := rm.redisCmdable.Incr(redisCmdData.key).Result()
					fmt.Println("REDIS_INCR key[%s] res:%v", redisCmdData.key, res)
					if err != nil {
						redisCmdData.replyChan <- &RedisResultS{Ok: false, Result: err}
					} else {
						redisCmdData.replyChan <- &RedisResultS{Ok: true, Result: res}
					}
				case REDIS_INCRBY:
					res, err := rm.redisCmdable.IncrBy(redisCmdData.key, redisCmdData.value.(int64)).Result()
					fmt.Println("REDIS_INCRBY key[%s] res:%v", redisCmdData.key, res)
					if err != nil {
						redisCmdData.replyChan <- &RedisResultS{Ok: false, Result: err}
					} else {
						redisCmdData.replyChan <- &RedisResultS{Ok: true, Result: res}
					}
				case REDIS_PUBLISH:
					res, err := rm.redisCmdable.Publish(redisCmdData.key, redisCmdData.value).Result()
					if err != nil {
						redisCmdData.replyChan <- &RedisResultS{Ok: false, Result: err}
					} else {
						redisCmdData.replyChan <- &RedisResultS{Ok: true, Result: res}
					}

				default:
					err := fmt.Errorf("Cmd[%s] does not support", redisCmdData.cmd.String())
					fmt.Println(err.Error())
					redisCmdData.replyChan <- &RedisResultS{Ok: false, Result: err}
				}
			}
		case <-rm.redisExitChan:
			break L

		}
	}
	//logs.Warn("redisCmdPerformRoutine Exit")
	return
}

func (rm *RedisMgr) GetClusterClient() *redis.ClusterClient {
	return rm.redisClusterClient
}

func (rm *RedisMgr) GetClient() *redis.Client {
	return rm.redisClient
}


func (rm *RedisMgr) Stop() {
	close(rm.redisExitChan)
	rm.redisRoutineWG.Wait()
	if rm.isCluster {
		rm.redisClusterClient.Close()
	} else {
		rm.redisClient.Close()
	}
}


func (rm *RedisMgr) StringGetEx(key string) (interface{}, error) {
	value, err := rm.StringGet(key)
	if err != nil && err.Error() == "redis: nil" {
		return "", nil
	}
	if err != nil {
		return nil, err
	}
	return value, nil
}

func (rm *RedisMgr) StringGet(key string) (interface{}, error) {
	reply := make(chan *RedisResultS)
	rm.redisCmdChan <- &RedisCommandData{
		cmd:       REDIS_GET,
		key:       key,
		replyChan: reply,
	}
	return rm.waitResult(reply, key)
}

func (rm *RedisMgr) StringSet(key string, value []byte) error {
	reply := make(chan *RedisResultS)
	rm.redisCmdChan <- &RedisCommandData{
		cmd:       REDIS_SET,
		key:       key,
		value:     value,
		replyChan: reply,
	}
	_, err := rm.waitResult(reply, key)
	return err
}

// 0: key不存在，> 0 删除成功
func (rm *RedisMgr) DelKey(key string) (int64, error) {
	reply := make(chan *RedisResultS)
	rm.redisCmdChan <- &RedisCommandData{
		cmd:       REDIS_DEL,
		key:       key,
		replyChan: reply,
	}

	hValue, err := rm.waitResult(reply, key)
	if err == nil {
		return hValue.(int64), nil
	}

	return 0, err
}

func (rm *RedisMgr) StringSetNX(key string, value []byte, ttl int) error {
	reply := make(chan *RedisResultS)
	rm.redisCmdChan <- &RedisCommandData{
		cmd:       REDIS_SETNX,
		key:       key,
		value:     value,
		ttl:       ttl,
		replyChan: reply,
	}
	_, err := rm.waitResult(reply, key)
	return err
}

func (rm *RedisMgr) InterfaceSet(key string, value interface{}, ttl int) error {
	reply := make(chan *RedisResultS)
	rm.redisCmdChan <- &RedisCommandData{
		cmd:       REDIS_SET,
		key:       key,
		value:     value,
		ttl:       ttl,
		replyChan: reply,
	}
	_, err := rm.waitResult(reply, key)
	return err
}

func (rm *RedisMgr) ListSet(key string, value []string) error {
	reply := make(chan *RedisResultS)
	rm.redisCmdChan <- &RedisCommandData{
		cmd:       REDIS_LSET,
		key:       key,
		value:     value,
		replyChan: reply,
	}
	_, err := rm.waitResult(reply, key)
	return err
}

func (rm *RedisMgr) ListGet(key string) ([]string, error) {
	reply := make(chan *RedisResultS)
	rm.redisCmdChan <- &RedisCommandData{
		cmd:       REDIS_LGET,
		key:       key,
		replyChan: reply,
	}
	lValue, err := rm.waitResult(reply, key)
	if err == nil {
		return lValue.([]string), nil
	}
	return nil, err
}

func (rm *RedisMgr) ListRPush(key string, value []string) (int, error) {
	reply := make(chan *RedisResultS)
	rm.redisCmdChan <- &RedisCommandData{
		cmd:       REDIS_LRPUSH,
		key:       key,
		value:     value,
		replyChan: reply,
	}

	hValue, err := rm.waitResult(reply, key)
	if err == nil {
		if hValue == nil {
			return 0, fmt.Errorf("value is nil")
		} else {
			return hValue.(int), nil
		}
	}

	return 0, err
}

func (rm *RedisMgr) ListRPushVariable(key string, value ...string) (int, error) {
	reply := make(chan *RedisResultS)
	rm.redisCmdChan <- &RedisCommandData{
		cmd:       REDIS_LRPUSH,
		key:       key,
		value:     value,
		replyChan: reply,
	}

	hValue, err := rm.waitResult(reply, key)
	if err == nil {
		if hValue == nil {
			return 0, fmt.Errorf("value is nil")
		} else {
			return hValue.(int), nil
		}
	}

	return 0, err
}

func (rm *RedisMgr) ListLPushVariable(key string, value ...string) (int, error) {
	reply := make(chan *RedisResultS)
	rm.redisCmdChan <- &RedisCommandData{
		cmd:       REDIS_LLPUSH,
		key:       key,
		value:     value,
		replyChan: reply,
	}

	hValue, err := rm.waitResult(reply, key)
	if err == nil {
		if hValue == nil {
			return 0, fmt.Errorf("value is nil")
		} else {
			return hValue.(int), nil
		}
	}

	return 0, err
}

func (rm *RedisMgr) ListRem(key string, v string, count int) (int, error) {
	reply := make(chan *RedisResultS)
	rm.redisCmdChan <- &RedisCommandData{
		cmd:       REDIS_LREM,
		key:       key,
		value:     strconv.Itoa(count),
		args:      []interface{}{v},
		replyChan: reply,
	}
	delCount, err := rm.waitResult(reply, key)
	if err == nil {
		return delCount.(int), nil
	}
	return 0, err
}

func (rm *RedisMgr) ListTrim(key string, start, stop int) error {
	reply := make(chan *RedisResultS)
	rm.redisCmdChan <- &RedisCommandData{
		cmd:       REDIS_LTRIM,
		key:       key,
		args:      []interface{}{start, stop},
		replyChan: reply,
	}
	_, err := rm.waitResult(reply, key)
	return err
}

func (rm *RedisMgr) ListLen(key string) (int, error) {
	reply := make(chan *RedisResultS)
	rm.redisCmdChan <- &RedisCommandData{
		cmd:       REDIS_LLEN,
		key:       key,
		replyChan: reply,
	}

	len, err := rm.waitResult(reply, key)
	if err == nil {
		return len.(int), nil
	}

	return 0, err
}

func (rm *RedisMgr) ListLPop(key string) (string, error) {
	reply := make(chan *RedisResultS)
	rm.redisCmdChan <- &RedisCommandData{
		cmd:       REDIS_LPOP,
		key:       key,
		replyChan: reply,
	}

	value, err := rm.waitResult(reply, key)
	if err == nil {
		return value.(string), nil
	}

	return "", err
}

func (rm *RedisMgr) HashMSet(key string, hashV map[string]interface{}) error {
	reply := make(chan *RedisResultS)
	rm.redisCmdChan <- &RedisCommandData{
		cmd:       REDIS_HMSET,
		key:       key,
		value:     hashV,
		replyChan: reply,
	}
	_, err := rm.waitResult(reply, key)
	return err
}

func (rm *RedisMgr) HashSet(key string, fieldKey string, value interface{}) error {
	reply := make(chan *RedisResultS)
	rm.redisCmdChan <- &RedisCommandData{
		cmd:       REDIS_HSET,
		key:       key,
		fieldKey:  fieldKey,
		value:     value,
		replyChan: reply,
	}
	_, err := rm.waitResult(reply, key)
	return err
}

func (rm *RedisMgr) HashGetAll(key string) (map[string]string, error) {
	reply := make(chan *RedisResultS)
	rm.redisCmdChan <- &RedisCommandData{
		cmd:       REDIS_HGETALL,
		key:       key,
		replyChan: reply,
	}
	hValue, err := rm.waitResult(reply, key)
	if err == nil {
		return hValue.(map[string]string), nil
	}
	return nil, err
}

func (rm *RedisMgr) HashGet(key string, fieldKey string) (interface{}, error) {
	reply := make(chan *RedisResultS)
	rm.redisCmdChan <- &RedisCommandData{
		cmd:       REDIS_HGET,
		key:       key,
		fieldKey:  fieldKey,
		replyChan: reply,
	}
	hValue, err := rm.waitResult(reply, key)
	if err == nil {
		return hValue, nil
	}
	return nil, err
}

func (rm *RedisMgr) HashDel(key string, fields ...string) (int64, error) {
	reply := make(chan *RedisResultS)
	rm.redisCmdChan <- &RedisCommandData{
		cmd:       REDIS_HDEL,
		key:       key,
		value:     fields,
		replyChan: reply,
	}
	delCount, err := rm.waitResult(reply, key)
	if err == nil {
		return delCount.(int64), nil
	}
	return 0, err
}

// 0: 不存在 1: 存在
func (rm *RedisMgr) Exists(key string) (int64, error) {
	reply := make(chan *RedisResultS)
	rm.redisCmdChan <- &RedisCommandData{
		cmd:       REDIS_EXISTS,
		key:       key,
		replyChan: reply,
	}
	hValue, err := rm.waitResult(reply, key)
	if err == nil {
		return hValue.(int64), nil
	}
	return 0, err
}

func (rm *RedisMgr) ScriptLoad(script []byte) (interface{}, error) {
	reply := make(chan *RedisResultS)
	rm.redisCmdChan <- &RedisCommandData{
		cmd:       REDIS_SCRIPT_LOAD,
		value:     script,
		replyChan: reply,
	}
	return rm.waitResult(reply, "script")
}

func (rm *RedisMgr) Evalsha(args []interface{}) (interface{}, error) {
	reply := make(chan *RedisResultS)
	rm.redisCmdChan <- &RedisCommandData{
		cmd:       REDIS_EVAlSHA,
		args:      args,
		replyChan: reply,
	}
	return rm.waitResult(reply, "script")
}

func (rm *RedisMgr) Incr(key string) (int64, error) {
	reply := make(chan *RedisResultS)
	rm.redisCmdChan <- &RedisCommandData{
		cmd:       REDIS_INCR,
		key:       key,
		replyChan: reply,
	}

	hValue, err := rm.waitResult(reply, key)
	if err == nil {
		if hValue == nil {
			return 0, fmt.Errorf("value is nil")
		} else {
			return hValue.(int64), nil
		}
	}

	return 0, err
}
func (rm *RedisMgr) IncrBy(key string, amount int64) (int64, error) {
	reply := make(chan *RedisResultS)
	rm.redisCmdChan <- &RedisCommandData{
		cmd:   REDIS_INCRBY,
		key:   key,
		value: amount,

		replyChan: reply,
	}

	hValue, err := rm.waitResult(reply, key)
	if err == nil {
		if hValue == nil {
			return 0, fmt.Errorf("value is nil")
		} else {
			return hValue.(int64), nil
		}
	}

	return 0, err
}

func (rm *RedisMgr) ZADD(key, fieldkey string, score float64) (int64, error) {
	reply := make(chan *RedisResultS)
	rm.redisCmdChan <- &RedisCommandData{
		cmd:       REDIS_ZADD,
		key:       key,
		fieldKey:  fieldkey,
		value:     score,
		replyChan: reply,
	}

	hValue, err := rm.waitResult(reply, key)
	if err == nil {
		if hValue == nil {
			return 0, fmt.Errorf("value is nil")
		} else {
			return hValue.(int64), nil
		}
	}

	return 0, err
}

func (rm *RedisMgr) ZIncrBy(key, member string, increment int) (int, error) {
	reply := make(chan *RedisResultS)
	rm.redisCmdChan <- &RedisCommandData{
		cmd:       REDIS_ZINCRBY,
		key:       key,
		args:      []interface{}{strconv.Itoa(increment), member},
		replyChan: reply,
	}

	hValue, err := rm.waitResult(reply, key)
	if err == nil {
		if hValue == nil {
			return 0, fmt.Errorf("value is nil")
		} else {
			return hValue.(int), nil
		}
	}

	return 0, err
}

func (rm *RedisMgr) ZScore(key, member string) (int, error) {
	reply := make(chan *RedisResultS)
	rm.redisCmdChan <- &RedisCommandData{
		cmd:       REDIS_ZSCORE,
		key:       key,
		args:      []interface{}{member},
		replyChan: reply,
	}

	hValue, err := rm.waitResult(reply, key)
	if err == nil {
		if hValue == nil {
			return 0, fmt.Errorf("member not exist")
		} else {
			return hValue.(int), nil
		}
	}

	return 0, err
}

func (rm *RedisMgr) ZRemRangeByRank(key string, start, stop int64) (int64, error) {
	reply := make(chan *RedisResultS)
	rm.redisCmdChan <- &RedisCommandData{
		cmd:       REDIS_ZREMRANGEBYRANK,
		key:       key,
		args:      []interface{}{start, stop},
		replyChan: reply,
	}

	hValue, err := rm.waitResult(reply, key)
	if err == nil {
		if hValue == nil {
			return 0, nil
		} else {
			return hValue.(int64), nil
		}
	}

	return 0, err
}

func (rm *RedisMgr) ZRem(key string, members ...string) (int64, error) {
	reply := make(chan *RedisResultS)
	rm.redisCmdChan <- &RedisCommandData{
		cmd:       REDIS_ZREM,
		key:       key,
		value:     members,
		replyChan: reply,
	}

	hValue, err := rm.waitResult(reply, key)
	if err == nil {
		if hValue == nil {
			return 0, nil
		} else {
			return hValue.(int64), nil
		}
	}

	return 0, err
}

func (rm *RedisMgr) ZReverseRank(key string, member string) (int, error) {
	reply := make(chan *RedisResultS)
	rm.redisCmdChan <- &RedisCommandData{
		cmd:       REDIS_ZRRANK,
		key:       key,
		value:     member,
		replyChan: reply,
	}

	hValue, err := rm.waitResult(reply, key)
	if err == nil {
		if hValue == nil {
			return 0, nil
		} else {
			return hValue.(int), nil
		}
	}

	return 0, err
}
func (rm *RedisMgr) ZRangeByScore(key string, by redis.ZRangeBy) ([]string, error) {
	reply := make(chan *RedisResultS)
	rm.redisCmdChan <- &RedisCommandData{
		cmd:       REDIS_ZRANGEBYSCORE,
		key:       key,
		value:     by,
		replyChan: reply,
	}

	hValue, err := rm.waitResult(reply, key)
	if err == nil {
		if hValue == nil {
			return nil, nil
		} else {
			return hValue.([]string), nil
		}
	}

	return nil, err
}

func (rm *RedisMgr) Expire(key string, seconds int) (bool, error) {
	reply := make(chan *RedisResultS)
	rm.redisCmdChan <- &RedisCommandData{
		cmd:       REDIS_EXPIRE,
		key:       key,
		value:     time.Duration(seconds) * time.Second,
		replyChan: reply,
	}

	ret, err := rm.waitResult(reply, key)
	if err == nil {
		return ret.(bool), err
	}
	return false, err
}

func (rm *RedisMgr) Pipelined(fn redisPipelineFn) ([]redis.Cmder, error) {
	reply := make(chan *RedisResultS)
	rm.redisCmdChan <- &RedisCommandData{
		cmd:       REDIS_PIPELINE,
		key:       "Pipelined",
		value:     fn,
		replyChan: reply,
	}

	hValue, err := rm.waitResult(reply, "Pipelined")
	if err == nil {
		return hValue.([]redis.Cmder), nil
	}

	return nil, err
}

func (rm *RedisMgr) DBSize() (int64, error) {
	reply := make(chan *RedisResultS)
	rm.redisCmdChan <- &RedisCommandData{
		cmd:       REDIS_DBSIZE,
		replyChan: reply,
	}

	hValue, err := rm.waitResult(reply, "DBSize")
	if err == nil {
		return hValue.(int64), nil
	}

	return 0, err
}

func (rm *RedisMgr) Scan(cursor uint64, match string, count int64) ([]string, uint64, error) {
	reply := make(chan *RedisResultS)
	rm.redisCmdChan <- &RedisCommandData{
		cmd:       REDIS_SCAN,
		args:      []interface{}{cursor, match, count},
		replyChan: reply,
	}

	hValue, err := rm.waitResult(reply, "Scan")
	if err == nil {
		scanRes := hValue.(*RedisScanResult)
		return scanRes.keys, scanRes.cursor, nil
	}

	return nil, 0, err
}

func (rm *RedisMgr) ClusterScan(masterNodeCmdable redis.Cmdable, cursor uint64, match string, count int64) ([]string, uint64, error) {
	reply := make(chan *RedisResultS)
	rm.redisCmdChan <- &RedisCommandData{
		cmd:       REDIS_CLUSTERSCAN,
		args:      []interface{}{masterNodeCmdable, cursor, match, count},
		replyChan: reply,
	}

	hValue, err := rm.waitResult(reply, "Scan")
	if err == nil {
		scanRes := hValue.(*RedisScanResult)
		return scanRes.keys, scanRes.cursor, nil
	}

	return nil, 0, err
}

func (rm *RedisMgr) SAdd(key string, members ...interface{}) (int64, error) {
	reply := make(chan *RedisResultS)
	rm.redisCmdChan <- &RedisCommandData{
		cmd:       REDIS_SADD,
		key:       key,
		args:      members,
		replyChan: reply,
	}

	hValue, err := rm.waitResult(reply, key)
	if err == nil {
		if hValue == nil {
			return 0, nil
		} else {
			return hValue.(int64), nil
		}
	}

	return 0, err
}

func (rm *RedisMgr) SRem(key string, members ...interface{}) (int64, error) {
	reply := make(chan *RedisResultS)
	rm.redisCmdChan <- &RedisCommandData{
		cmd:       REDIS_SREM,
		key:       key,
		args:      members,
		replyChan: reply,
	}

	hValue, err := rm.waitResult(reply, key)
	if err == nil {
		if hValue == nil {
			return 0, nil
		} else {
			return hValue.(int64), nil
		}
	}
	return 0, err
}

func (rm *RedisMgr) SIsMember(key string, member interface{}) (bool, error) {
	reply := make(chan *RedisResultS)
	rm.redisCmdChan <- &RedisCommandData{
		cmd:       REDIS_SISMEMBER,
		key:       key,
		value:     member,
		replyChan: reply,
	}

	hValue, err := rm.waitResult(reply, key)
	if err == nil {
		return hValue.(bool), nil
	}

	return false, err
}

func (rm *RedisMgr) SCARD(key string) (int64, error) {
	reply := make(chan *RedisResultS)
	rm.redisCmdChan <- &RedisCommandData{
		cmd:       REDIS_SCARD,
		key:       key,
		replyChan: reply,
	}

	hValue, err := rm.waitResult(reply, key)
	if err == nil {
		return hValue.(int64), nil
	}

	return 0, err
}

func (rm *RedisMgr) SMEMBERS(key string) ([]string, error) {
	reply := make(chan *RedisResultS)
	rm.redisCmdChan <- &RedisCommandData{
		cmd:       REDIS_SMEMBERS,
		key:       key,
		replyChan: reply,
	}

	hValue, err := rm.waitResult(reply, key)
	if err == nil {
		return hValue.([]string), nil
	}

	return nil, err
}

func (rm *RedisMgr) Publish(key string, msg string) error {
	reply := make(chan *RedisResultS)
	rm.redisCmdChan <- &RedisCommandData{
		cmd:       REDIS_PUBLISH,
		key:       key,
		value:     msg,
		replyChan: reply,
	}

	_, err := rm.waitResult(reply, key)
	return err
}

func (rm *RedisMgr) waitResult(reply <-chan *RedisResultS, tag string) (interface{}, error) {
	select {
	case redisRes, ok := <-reply:
		if ok {
			if redisRes.Ok {
				return redisRes.Result, nil
			} else {
				return nil, redisRes.Result.(error)
			}
		}
	case <-time.After(2 * time.Second):
		return nil, ErrTimeOut(tag)
	}
	return nil, ErrTimeOut(tag)
}

func main () {
	//var redisMgrr *RedisMgr
	//var redisMgrLock *RedisMgr
	redisMgrr := RedisMgr{}
	redisMgrLock := RedisMgr{}
	InitRedis(&redisMgrr,&redisMgrLock)
	fmt.Println("redisMgrr：",redisMgrr) //--------①

	redisMgrr.StringSet("keyyy",[]byte("中文字符"))
	ret, _ := redisMgrr.StringGet("keyyy")
	fmt.Printf("看看ret:",ret)
}
