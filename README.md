#! /usr/bin/python
import cv2
import numpy as np
import os.path as path
import time
 
def center(points):
    #计算给定矩阵的质心
    x = (points[0][0] + points[1][0] + points[2][0] + points[3][0]) / 4
    y = (points[0][1] + points[1][1] + points[2][1] + points[3][1]) / 4
    return np.array([np.float32(x), np.float32(y)], np.float32)
 

 
class Pedestrian():
  #一个行人由一个ROI和一个卡尔曼滤波器组成。所以创建一个行人类来保持对象状态   
  def __init__(self, frame, track_window):
    print('只初始化一次')
    x,y,w,h = track_window
    self.track_window = track_window #要跟踪目标的初始位置矩形框
    self.roi = cv2.cvtColor(frame[y:y+h, x:x+w], cv2.COLOR_BGR2HSV) ##crop frame ==> frame[ymin:ymax, xmin:xmax]
    mask = cv2.inRange(self.roi, np.array((0., 60.,32.)), np.array((180.,255.,255.)))
    #计算ROI的直方图
    self.roi_hist = cv2.calcHist([self.roi], [0], mask, [180], [0, 180])
    cv2.normalize(self.roi_hist, self.roi_hist, 0, 255, cv2.NORM_MINMAX)
    self.term_crit = ( cv2.TERM_CRITERIA_EPS | cv2.TERM_CRITERIA_COUNT, 10, 1 )
 
    '''
    #设置卡尔曼滤波器，不弄这个了，没什么用
    self.kalman = cv2.KalmanFilter(4,2)
    self.kalman.measurementMatrix = np.array([[1,0,0,0],[0,1,0,0]],np.float32)
    self.kalman.transitionMatrix = np.array([[1,0,1,0],[0,1,0,1],[0,0,1,0],[0,0,0,1]],np.float32)
    self.kalman.processNoiseCov = np.array([[1,0,0,0],[0,1,0,0],[0,0,1,0],[0,0,0,1]],np.float32) * 0.03
    self.measurement = np.array((2,1), np.float32) 
    self.prediction = np.zeros((2,1), np.float32)
    self.center = None
    '''
     

 
  def update(self, frame):
    hsv = cv2.cvtColor(frame, cv2.COLOR_BGR2HSV)
    back_project = cv2.calcBackProject([hsv],[0], self.roi_hist,[0,180],1)    
    ret, self.track_window = cv2.CamShift(back_project, self.track_window, self.term_crit)
    '''
    OpenCV没有函数能直接从轮廓信息中计算出最小矩形顶点的坐标。所以需要计算出最小矩形区域，然后计算这个矩形的顶点。
    # 计算包围目标的最小矩形区域
    rect = cv2.minAreaRect(counter)

    # 计算矩形的 4 点坐标，返回结果为float数据类型
    box = cv2.boxPoints(rect)
    
　  # 将float类型转为 int，在这里最好用成int32 / int64而不是int0
    box =np.int0(box)
    '''
    pts = cv2.boxPoints(ret)
    pts = np.int0(pts) #类似[[1,3],[2, 3],[3,4],[5, 6]]
    #self.center = center(pts)
    cv2.polylines(frame,[pts],True, (255,255,0),10) #画面上青色的那个框框
    #time.sleep(20)
    
    
    '''
    #根据测量更新预测状态。
    self.kalman.correct(self.center)
    prediction = self.kalman.predict()
    cv2.circle(frame, (int(prediction[0]), int(prediction[1])), 4, (0, 0, 255), -1)
    '''
     
 
def main():
  camera = cv2.VideoCapture("../../img/bike.avi")
  #camera = cv2.VideoCapture(0)
   
  #KNN background subtractor
  bs = cv2.createBackgroundSubtractorKNN()
   
  cv2.namedWindow("zhuchuankou")
   
  pedestrians = {}
  findGenZhongMuBiao = False
  frames_num = 0
  genzhong_obj = '';
   
  while True:
    grabbed, frame = camera.read()
    if (grabbed is False):
      print ("failed to grab frame.")
      break
 
    if not findGenZhongMuBiao:
        fgmask = bs.apply(frame)
 
    if not findGenZhongMuBiao and frames_num > 50: #过滤掉前50帧
    #if 1: 
        #通过对前景掩模采用膨胀和腐蚀的方法来识别斑点及其周围边框
        th = cv2.threshold(fgmask.copy(), 127, 255, cv2.THRESH_BINARY)[1]
        th = cv2.erode(th, cv2.getStructuringElement(cv2.MORPH_ELLIPSE, (3,3)), iterations = 2)
        dilated = cv2.dilate(th, cv2.getStructuringElement(cv2.MORPH_ELLIPSE, (8,3)), iterations = 2)
        cv2.imshow("chuangkou", dilated)     
        image, contours, hier = cv2.findContours(dilated, cv2.RETR_EXTERNAL, cv2.CHAIN_APPROX_SIMPLE)
 
    
        for c in contours:
            print('看看轮廓面积：',cv2.contourArea(c),'--当前帧数：',frames_num)
            if cv2.contourArea(c) > 260:
                (x,y,w,h) = cv2.boundingRect(c)
                
                #手动微调，精准框出对象，别问我10怎么来的，我随便写的
                w = w+10
                h = h+10
                x = x -10
                y = y -10

                cv2.rectangle(frame, (x,y), (x+w, y+h), (255, 0, 255), 1)
                genzhong_obj = Pedestrian(frame, (x,y,w,h)) #对选中的运动物体轮廓进行实例化
                findGenZhongMuBiao = True
                break
    
    print('找到跟踪目标了吗？：',findGenZhongMuBiao)
    if genzhong_obj:
        genzhong_obj.update(frame)

    
    frames_num += 1
    cv2.imshow("zhuchuankou", frame)
    if cv2.waitKey(110)  & 0xFF == ord('q'):
        break
  camera.release()
  cv2.destroyAllWindows()
 
if __name__ == "__main__":
  main()
