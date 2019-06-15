'''
云台人脸追踪
解决方案：
使用OpenCV的人脸检测的API获取人脸在画面中的位置，根据人脸位置距离画面中心的x轴与y轴的偏移量(offset) ，通过P比例控制(PID控制中最简单的一种)控制二自由度云台上臂与下臂的旋转角度，将角度信息通过串口通信发送给Arduino单片机解析执行对应的操作，从而使得人脸尽可能处在画面的正中间。

这个时候我们就需要参照人脸中心在X轴方向的偏移量 offset
画面中心X坐标， 如果画面分辨率为800×600 ，那么对于x轴来讲x_center = 800/2 = 400 .
实际值(Real) 指的就是人脸矩形区域中心在画面中坐标的X轴取值。
偏移量(Offset)是指的人脸中心偏移画面中心的值。
Offset = Real - x_center

根据x_offset 的正负可以得到舵机角度的旋转方向（角度增大还是减小）
需要在偏移量与舵机角度的增量之间建立某种联系,这种关系就是线性关系y=k*x。这里我们定义一个比例系数(Kp)来描述这段关系：
delta_degree = Kp * x_offset
新的舵机角度公式为：
新的舵机角度 = 旧的舵机角度 + 角度增量
new_degree = old_degree + delta_degree
'''




# -*- coding:utf-8 -*-
import cv2
import time
import RPi.GPIO as GPIO

ServoPinA = 12
ServoPinB = 6

GPIO.setmode(GPIO.BCM)
GPIO.setwarnings(False)

#记录的上一次舵机所在的角度
last_btm_degree = 90
last_top_degree = 0

#控制舵机旋转的比例系数
btm_kp = 5
top_kp = 5 

#设定偏移量死区范围
#解决目标在画面中心附近舵机来回摆动的问题，如果偏移量小于一定的数值舵机就不动。
offset_dead_block = 0.1

def car_init():
    global pwm_servoA
    global pwm_servoB
    GPIO.setup(ServoPinA, GPIO.OUT)
    GPIO.setup(ServoPinB, GPIO.OUT)
    pwm_servoA = GPIO.PWM(ServoPinA, 50)
    pwm_servoB = GPIO.PWM(ServoPinB, 50)
    pwm_servoA.start(0)
    pwm_servoB.start(0)
    # 舵机角度初始化
    xuanzhuang(last_btm_degree, last_top_degree)

#舵机转动
def xuanzhuang(bottom_degree, top_degree):
    print('假装在旋转')
    pwm_servoA.ChangeDutyCycle(((0.5+(1/90)*bottom_degree)/20)*100)
    time.sleep(1)
    pwm_servoA.ChangeDutyCycle(0)

    pwm_servoB.ChangeDutyCycle(((0.5+(1/90)*top_degree)/20)*100)
    time.sleep(1)
    pwm_servoB.ChangeDutyCycle(0)





def btm_servo_control(offset_x):
    '''
    底部舵机的比例控制
    这里舵机使用开环控制
    '''
    global offset_dead_block
    global btm_kp
    global last_btm_degree
    
    if abs(offset_x) < offset_dead_block:
       offset_x = 0

    delta_degree = offset_x * btm_kp
    # 计算得到新的底部舵机角度
    next_btm_degree = last_btm_degree + delta_degree
    # 添加边界检测
    if next_btm_degree < 0:
        next_btm_degree = 0
    elif next_btm_degree > 180:
        next_btm_degree = 180
    
    return int(next_btm_degree)

def top_servo_control(offset_y):
    global offset_dead_block
    global top_kp
    global last_top_degree

    if abs(offset_y) < offset_dead_block:
        offset_y = 0

    delta_degree = offset_y * top_kp
    next_top_degree = last_top_degree + delta_degree
    
    if next_top_degree < 0:
        next_top_degree = 0
    elif next_top_degree > 180:
        next_top_degree = 180
    
    return int(next_top_degree)



def calculate_offset(frame_width, frame_height, face):
    '''
    计算脸框中心相对画面中心的偏移量归一化在[-1,1]之间的值
    因为不同的摄像头拍出的画面分辨率不同，所以偏移量必须归一化
    '''
    (x, y, w, h) = face #这个x、y是脸框左上角顶点相对画面左上角顶点为原点的坐标
    face_x = float(x + w/2.0) #脸框中心点x坐标（相对画面左上角顶点）
    face_y = float(y + h/2.0)
    '''
    计算脸框中心相对画面中心在X轴上的偏移量
    offset_x = face_x - frame_width/2
    这个偏移量所占画面宽度的比例为：bili = offset_x/frame_width
    这个比例乘以偏移量取值范围长度（-1到1长度是2），得到偏移量归一化在归一化范围[-1,1]的值
    '''
    offset_x = float(face_x / frame_width - 0.5) * 2
    offset_y = float(face_y / frame_height - 0.5) * 2

    return (offset_x, offset_y)



def genzhong():
    #for Mac
    FaceCascade = cv2.CascadeClassifier("/usr/local/Cellar/opencv/3.4.3_2/share/OpenCV/haarcascades/haarcascade_frontalface_alt.xml")
    #for raspberryPi
    #FaceCascade = cv2.CascadeClassifier('/usr/local/share/OpenCV/haarcascades/haarcascade_frontalface_alt.xml')
    cv2.namedWindow('FaceDetect',flags=cv2.WINDOW_NORMAL | cv2.WINDOW_KEEPRATIO | cv2.WINDOW_GUI_EXPANDED)

    cap = cv2.VideoCapture(0)
    # 设置缓存区的大小
    cap.set(cv2.CAP_PROP_BUFFERSIZE,1)

    while cap.isOpened():
        ret, frame = cap.read()
        frame = cv2.flip(frame, 1)
        gray = cv2.cvtColor(frame, cv2.COLOR_BGR2GRAY)
        faces = FaceCascade.detectMultiScale(gray,scaleFactor=1.1,minNeighbors=5)
    
        if len(faces) == 0:
            face = None
        else :
            # 目前找的是画面中面积最大的人脸
            face =  max(faces, key=lambda face: face[2]*face[3])
            (x, y, w, h) = face
        if face is not None:
            cv2.rectangle(frame, (x, y), (x+w, y+h), (0, 255, 0), 4)

            frame_height, frame_width,_ = frame.shape
            (offset_x, offset_y) = calculate_offset(frame_width, frame_height, face)
            print("X轴偏移量：{} Y轴偏移量：{}".format(offset_x, offset_y))
            # 计算下一步舵机要转的角度
            next_btm_degree = btm_servo_control(offset_x)
            next_top_degree = top_servo_control(offset_y)
            print('底部舵机将要旋转到：{}度 顶部舵机将要旋转到：{}度'.format(next_btm_degree, next_top_degree))
            xuanzhuang(next_btm_degree, next_top_degree)
            # 更新角度值
            last_btm_degree = next_btm_degree
            last_top_degree = next_top_degree
        

        cv2.imshow('FaceDetect', frame)
    
        key = cv2.waitKey(1)
        if key == ord('q'):
            break

    cap.release()
    cv2.destroyAllWindows()

try:
    car_init()
    genzhong()
except KeyboardInterrupt:
    pass
print('即将退出')
pwm_servoA.stop()
pwm_servoB.stop()
GPIO.cleanup()
