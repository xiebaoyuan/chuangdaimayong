'''
带sim卡的终端设备必须作为客户端不能作为服务器。原因有二。1.不符合物联网设计规范。2.假如终端设备更换sim 卡，那么服务器ip地址也就丢失了，服务器都找不到了还怎么玩？
'''
import socket
import json
import os
import time
import array
import RPi.GPIO as GPIO

#电机引脚
AIN1 = 16
AIN2 = 20
BIN1 = 13
BIN2 = 19
ENA = 21
ENB = 26

#摄像头舵机引脚
ServoPinA = 12
ServoPinB = 6


GPIO.setmode(GPIO.BCM)
GPIO.setwarnings(False)

def car_init():
    global pwm_ENA
    global pwm_ENB
    global pwm_servoA
    global pwm_servoB
    GPIO.setup(ServoPinA, GPIO.OUT)
    GPIO.setup(ServoPinB, GPIO.OUT)
    GPIO.setup(ENA,GPIO.OUT,initial=GPIO.HIGH)
    GPIO.setup(AIN1,GPIO.OUT,initial=GPIO.LOW)
    GPIO.setup(AIN2,GPIO.OUT,initial=GPIO.LOW)
    GPIO.setup(ENB,GPIO.OUT,initial=GPIO.HIGH)
    GPIO.setup(BIN1,GPIO.OUT,initial=GPIO.LOW)
    GPIO.setup(BIN2,GPIO.OUT,initial=GPIO.LOW)
    #初始化pwm引脚,设置频率为2000hz
    pwm_ENA = GPIO.PWM(ENA, 2000)
    pwm_ENB = GPIO.PWM(ENB, 2000)
    pwm_servoA = GPIO.PWM(ServoPinA, 50)
    pwm_servoB = GPIO.PWM(ServoPinB, 50)
    pwm_ENA.start(0)
    pwm_ENB.start(0)
    pwm_servoA.start(0)
    pwm_servoB.start(0)

def carmotion_run():
    GPIO.output(AIN1, GPIO.HIGH)
    GPIO.output(AIN2, GPIO.LOW)
    GPIO.output(BIN1, GPIO.HIGH)
    GPIO.output(BIN2, GPIO.LOW)
    pwm_ENA.ChangeDutyCycle(50)
    pwm_ENB.ChangeDutyCycle(50)

def carmotion_back():
    GPIO.output(AIN1, GPIO.LOW)
    GPIO.output(AIN2, GPIO.HIGH)
    GPIO.output(BIN1, GPIO.LOW)
    GPIO.output(BIN2, GPIO.HIGH)
    pwm_ENA.ChangeDutyCycle(50)
    pwm_ENB.ChangeDutyCycle(50)

def carmotion_right():
    GPIO.output(AIN1, GPIO.HIGH)
    GPIO.output(AIN2, GPIO.LOW)
    GPIO.output(BIN1, GPIO.HIGH)
    GPIO.output(BIN2, GPIO.LOW)
    pwm_ENA.ChangeDutyCycle(0)
    pwm_ENB.ChangeDutyCycle(80)

def carmotion_left():
    GPIO.output(AIN1, GPIO.HIGH)
    GPIO.output(AIN2, GPIO.LOW)
    GPIO.output(BIN1, GPIO.HIGH)
    GPIO.output(BIN2, GPIO.LOW)
    pwm_ENA.ChangeDutyCycle(80)
    pwm_ENB.ChangeDutyCycle(0)

def carmotion_stop():
    GPIO.output(AIN1, GPIO.LOW)
    GPIO.output(AIN2, GPIO.LOW)
    GPIO.output(BIN1, GPIO.LOW)
    GPIO.output(BIN2, GPIO.LOW)
    pwm_ENA.ChangeDutyCycle(50)
    pwm_ENB.ChangeDutyCycle(50)


def xuanzhuang(pwm_servo_obj,pos):
    pwm_servo_obj.ChangeDutyCycle(((0.5+(1/90)*pos)/20)*100)
    time.sleep(1)
    pwm_servo_obj.ChangeDutyCycle(0)

def genzhong(data):
    servoa_degree = data[4]
    servob_degree = data[5]
    turn_right = data[1]
    turn_left = data[2]
    move_run = data[3]
    servoMotion = True
    if servoa_degree == 10000 or servob_degree == 10000:
        servoMotion = False
    if servoMotion:
        if servoa_degree < 0:
            servoa_degree = 0
        elif servoa_degree >180 :
            servoa_degree = 180
        if servob_degree < 0 :
            servob_degree = 0
        elif servob_degree > 180:
            servob_degree = 180

        #我买了一套新的舵机，参数略有变化，90变成135了
        pwm_servoA.ChangeDutyCycle(((0.5+(1/135)*servoa_degree)/20)*100)
        time.sleep(0.02) #20ms一个周期信号
        pwm_servoA.ChangeDutyCycle(0)
        time.sleep(0.2)
    
        #不要同时给两路pwm信号，我怕树莓派会被烧掉
        pwm_servoB.ChangeDutyCycle(((0.5+(1/135)*servob_degree)/20)*100)
        time.sleep(0.02)
        pwm_servoB.ChangeDutyCycle(0)
        time.sleep(0.2)

    #转向
    if turn_right :
        carmotion_right()
        time.sleep(.5)
        carmotion_stop()
    elif turn_left :
        carmotion_left()
        time.sleep(.5)
        carmotion_stop()

    #前进
    if move_run :
        carmotion_run()
        time.sleep(.5)
        carmotion_stop()

class sockClient(object):
    def __init__(self):
        self.client = socket.socket()
    def connect(self,ip,port):
        self.client.connect((ip,port))
    def zhuce(self):
        while True:
            #msg_str = input('输入发送给服务器的消息:').strip()
            #if len(msg_str) == 0 : continue
            time.sleep(5)
            msg_arr = [74,39,15,13] #注册包
            self.client.send(json.dumps(msg_arr).encode('utf-8'))
            server_res = self.client.recv(1024)
            temp = array.array('B',server_res) #就是一个数组。接收转化自服务器那端发来的sock.write(new Buffer([0x4a, 1]))
            if temp[0] == 74: #说明在服务器注册成功，无需再发送注册包了。
                break
        self.jiaohu() 
    def jiaohu(self):
        while True:
            jiaohudata = self.client.recv(1024)
            jiaohudata = array.array('B',jiaohudata)
            print('收到服务器下发数据：',jiaohudata)
            if not jiaohudata:
                print('服务器主动断开连接了')
                break

            if jiaohudata[0] == 60: #小车运动
                if jiaohudata[3] == 1:
                    carmotion_run()
                elif jiaohudata[3] == 2:
                    carmotion_back()
                elif jiaohudata[3] == 3:
                    carmotion_left()
                elif jiaohudata[3] == 4:
                    carmotion_right()
                elif jiaohudata[3] == 0:
                    carmotion_stop()
            elif jiaohudata[0] == 61: #摄像头舵机转动
                dushu = (jiaohudata[4] <<8) + jiaohudata[5]
                if dushu<=180 and dushu>=0 :
                    if jiaohudata[3] == 1:
                        xuanzhuang(pwm_servoA,dushu)
                    elif jiaohudata[3] == 2:
                        xuanzhuang(pwm_servoB,dushu)
            elif jiaohudata[0] == 62 : #主机发给小车的跟踪信号
                jiaohudata = jiaohudata[-6:]
                print('最新的指令：',jiaohudata)
                genzhong(jiaohudata)
            else:
                print('无法识别的指令：',jiaohudata)

try:
    car_init()
    carclient = sockClient()
    carclient.connect('123.207.18.153',6969)
    carclient.zhuce()
except KeyboardInterrupt:
    pass
pwm_ENA.stop()
pwm_ENB.stop()
pwm_servoA.stop()
pwm_servoB.stop()
GPIO.cleanup()
