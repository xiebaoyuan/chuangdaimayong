import cv2
import sys
 
def CatchUsbVideo(window_name, camera_idx):
    cv2.namedWindow(window_name)
    
    cap = cv2.VideoCapture(camera_idx)
    
    #for Mac
    #classfier = cv2.CascadeClassifier("/usr/local/Cellar/opencv/3.4.3_2/share/OpenCV/haarcascades/haarcascade_frontalface_alt.xml")
    #for raspberryPi
    classfier = cv2.CascadeClassifier('/usr/local/share/OpenCV/haarcascades/haarcascade_frontalface_alt.xml')
    
    color = (0, 255, 0)
        
    while cap.isOpened():
        ok, frame = cap.read()
        frame = cv2.flip(frame, 1)
        if not ok:            
            break  
 
        grey = cv2.cvtColor(frame, cv2.COLOR_BGR2GRAY)                 
        
        #1.2和2分别为图片缩放比例和需要检测的有效点数
        faceRects = classfier.detectMultiScale(grey, scaleFactor = 1.2, minNeighbors = 3, minSize = (32, 32))
        if len(faceRects) > 0:  #大于0则检测到人脸                                   
            for faceRect in faceRects:  #框出每一张人脸
                x, y, w, h = faceRect
                cv2.rectangle(frame, (x - 10, y - 10), (x + w + 10, y + h + 10), color, 2)
                        
        cv2.imshow(window_name, frame)        
        c = cv2.waitKey(10)
        if c & 0xFF == ord('q'):
            break        
    
    cap.release()
    cv2.destroyAllWindows() 
    
if __name__ == '__main__':
    if len(sys.argv) != 1:
        print("Usage:%s camera_id\r\n" % (sys.argv[0]))
    else:
        CatchUsbVideo("识别人脸区域", 0)
