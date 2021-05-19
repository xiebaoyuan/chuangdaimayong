# -*- coding:utf-8 -*-
import cv2
import time
import socket
import json
import array
import math
import numpy as np
from zhixiannihe import *


frame_height = 0
frame_width = 0
qianjingcishu = 0
guaijiaohengxianjuli = 10

def adjustDirection():
    global frame_height
    global frame_width
    global guaijiaohengxianjuli
    cv2.namedWindow('theRoad')

    #cap = cv2.VideoCapture(0)
    #使用树莓派的ip摄像头
    ip_camera_url = 'http://192.168.43.126:8080/?action=stream'
    cap = cv2.VideoCapture(ip_camera_url)
    cap.set(cv2.CAP_PROP_FOURCC,cv2.VideoWriter.fourcc('M','J','P','G') ) #设置摄像头读取视频编码格式
    cap.set(cv2.CAP_PROP_FRAME_WIDTH, 640)
    cap.set(cv2.CAP_PROP_FRAME_HEIGHT, 480)
    cap.set(cv2.CAP_PROP_BUFFERSIZE,1)

    last_cal_time = 0

    while cap.isOpened():
        ret, frame = cap.read()
        #frame = cv2.flip(frame, 1)
        frame_height = frame.shape[0]
        frame_width = frame.shape[1]
        if not ret:
            break
        face = None
        now = int(time.time())
        if (now-last_cal_time) > 2: #限定前后两次运算间隔2秒
            #开始检测
            pedestrian_obj = Pedestrian()
            backpro_img = pedestrian_obj.update(frame)
            ld = linedetector()
            #根据拐角横线距离来选择用哪种方式控制小车
            print('====>>>>>>>>>>>>>>>>>>拐角横线距离<<<<<<<<<<<<<<<<<<<<<<====',guaijiaohengxianjuli)
            if guaijiaohengxianjuli>0 and guaijiaohengxianjuli<180:
                bazi_kongzhi(ld,backpro_img,frame)
            elif guaijiaohengxianjuli>=180 and guaijiaohengxianjuli<200:
                print('事故频发地段，禁止左右转弯，直冲关卡')
                carclient.sendCommand(10000)

                #更新拐角横线距离
                ld.find_guaijiao_hengxian(backpro_img)
                if ld.guaijiao_hengxian:
                    guaijiaohengxianjuli = ld.guaijiao_hengxian['dist_to_top']
            elif guaijiaohengxianjuli > 200:
                print('把此前进行机器学习的结果拿来决策')
                carclient.sendCommand(40,True) #假装学习结果是左转
                guaijiaohengxianjuli = 10 #初始化拐角横线距离
                #还要初始化机器学习结果值

            last_cal_time = int(time.time())
            
        cv2.imshow('theRoad', frame)
    
        key = cv2.waitKey(1)
        if key == ord('q'):
            break

    cap.release()
    cv2.destroyAllWindows()


'''
不应让小车自己左右摇摆慢慢贴近拐角，首先拐角处道路复杂，有很多直线干扰，影响判断。其次，在拐角处
小车贴近左侧道路边缘跟贴近右侧道路边缘所处的情形相差甚远。最后最关键的，小车在太靠近拐角的地方
看不到拐角，只能看到一条横线，机器学习也无法决策。
解决方案：当小车在有八字道路情形中可以直线前进时，检测此时拐角的横线离小车的距离，并把此时的画面
交给机器学习，当这个距离到达一个阈值后，此后不应再使用机器视觉检测八字道路来决策小车行走方向，
而应该让小车直冲关卡(一直直线前进)，这时候不再看左右两侧道路而只看前方拐角的横线，当拐角的横线
离小车的距离再次到达另一个阈值时，用之前机器学习决策的结果来决定小车转向。
'''
def qianjing_xuexi(ld,backpro_img):
    global guaijiaohengxianjuli
    ld.find_guaijiao_hengxian(backpro_img)
    if ld.guaijiao_hengxian:
        guaijiaohengxianjuli = ld.guaijiao_hengxian['dist_to_top']
    print('把图片给机器学习')
    cv2.imshow('xuexitupian',backpro_img)


#当有检测到完整八字形道路时，根据八字的竖直中线位置调整小车位置使得小车位于道路中央，或者也可以根据中线的角度来控制小车左转/右转。
#当只检测到八字一撇/一捺时，根据一撇/一捺的角度来控制小车左转/右转。
def bazi_kongzhi(ld,backpro_img,frame):
    global qianjingcishu
    global guaijiaohengxianjuli
    xingtai_image = ld.xingtaichaozuo(backpro_img)
    ld.find_bazi_lines(xingtai_image)
    #print('八字道路：',ld.baZiLines)
    if ld.baZiLines.get('pie') and ld.baZiLines.get('na'):
        line_pie = ld.baZiLines.get('pie')
        line_na = ld.baZiLines.get('na')
        jiaodian = calJiaoDian(line_pie,line_na)
        zhongdian_x = calMidX(line_pie,line_na,frame_height)
        #jiaodian和zhongdian_x确定了八字道路的竖直中线
        chazhi_x = zhongdian_x[0]-jiaodian[0]
        print('中点横坐标减交点横坐标：',chazhi_x)
        cv2.line(frame, tuple(jiaodian), tuple(zhongdian_x), (255,0,255), 5)
        if chazhi_x > 60:
            carclient.sendCommand(-40)
        elif chazhi_x < -50:
            carclient.sendCommand(40)
        else:
            carclient.sendCommand(10000) #前进
            qianjing_xuexi(ld,backpro_img)
        qianjingcishu = 0
    elif ld.baZiLines.get('pie') or ld.baZiLines.get('na'):
        jiaodu = 10000
        if ld.baZiLines.get('pie'):
            jiaodu = ld.baZiLines.get('pie').get('jiaodu')
            #以下为考虑特殊干扰情况，比如明明道路在图片靠右侧，却下达指令让小车右转
            jiaodian_x = cal_jiaodian_X(ld.baZiLines.get('pie'),frame_height)
            if jiaodian_x > frame_width/2 :
                jiaodu = 40 #人为干预使之左转
        elif ld.baZiLines.get('na'):
            jiaodu = ld.baZiLines.get('na').get('jiaodu')
            jiaodian_x = cal_jiaodian_X(ld.baZiLines.get('na'),frame_height)
            if jiaodian_x < frame_width/2 :
                jiaodu = -40 #人为干预使之右转
        print('只得到一撇或一捺:',ld.baZiLines)
        if jiaodu > -20 and jiaodu < 20:
            print('这应该属不正常情况了，因为八字控制的时候不应该检测到拐角横线来控制啊')
            #即这种特殊情况：假如小车一直左摆右摆步履蹒跚地走到拐角前，也就是横线距离一直
            #得不到更新，也就是说略过了直冲关卡的阶段。那么此时应该还要去检测近乎水平的线段
            #更新拐角横线离相框上边的距离。
        else:
            carclient.sendCommand(jiaodu)
        ld.find_guaijiao_hengxian(backpro_img)
        if ld.guaijiao_hengxian:
            guaijiaohengxianjuli = ld.guaijiao_hengxian['dist_to_top']
        qianjingcishu = 0
    elif not ld.baZiLines.get('pie') and not ld.baZiLines.get('na'):
        #先尝试向前走两步，如果走两步之后还是None，那就停止
        if qianjingcishu < 2:
            #前进
            carclient.sendCommand(10000)
            qianjingcishu += 1
            ld.find_guaijiao_hengxian(backpro_img)
            if ld.guaijiao_hengxian:
                guaijiaohengxianjuli = ld.guaijiao_hengxian['dist_to_top']


def calJiaoDian(line1,line2):
    #求两条直线的交点坐标
    k1 = line1['k']
    b1 = line1['b']
    k2 = line2['k']
    b2 = line2['b']
    x=(b2-b1)/(k1-k2)
    y = k1*x + b1
    return [int(x),int(y)]

def calMidX(line1,line2,height):
    #求两条直线与图片底边的两个交点之间的中点
    b1 = line1['b']
    k1 = line1['k']
    b2 = line2['b']
    k2 = line2['k']
    x1 = (height-b1)/k1
    x2 = (height-b2)/k2
    return [int((x1+x2)/2),int(height)]

def cal_jiaodian_X(line,height):
    #求直线与相框底边的交点横坐标x
    b = line['b']
    k = line['k']
    x = (height-b)/k
    return int(x)

def test(img):
    global frame_height
    global frame_width
    frame_height = img.shape[0]
    frame_width = img.shape[1]
    pedestrian_obj = Pedestrian()
    backpro_img = pedestrian_obj.update(img)
    ld = linedetector()
    bazi_kongzhi(ld,backpro_img,img)
    cv2.waitKey(0)
    cv2.destroyAllWindows()


class sockClient(object):
    def __init__(self):
        self.client = socket.socket()
    def connect(self,ip,port):
        self.client.connect((ip,port))
    def sendCommand(self,zhixianjiaodu,dafudu=False):
        msg_arr = [62,0,0,0]
        value = 1
        if dafudu: #大幅度转弯
            value = 2
        if zhixianjiaodu > -60 and zhixianjiaodu < -20:
            msg_arr[1] = value 
            print('----------***右转***----------')
        elif zhixianjiaodu > 20 and zhixianjiaodu < 70:
            msg_arr[2] = value 
            print('----------***左转***----------')
        else:
            msg_arr[3] = 1
            print('----------***前进***----------')
        msg_arr.append(66) #水平舵机旋转角度
        msg_arr.append(66) #垂直舵机旋转角度
        #当两者都是66时，是约定舵机不动的暗号，不能用太大的数，会超出字节数
        self.client.send(json.dumps(msg_arr).encode('utf-8'))


if __name__ == "__main__":
    try:
        carclient = sockClient()
        carclient.connect('123.207.18.153',6969)
        #adjustDirection()
        test_image = cv2.imread('img/test_img18.jpg')
        test(test_image)
    except KeyboardInterrupt:
        pass
    print('即将退出')
