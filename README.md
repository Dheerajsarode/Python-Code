# Python-Code
# Drowsy Driving Detection &amp; Traffic Collision Information System With Obstacle Detection System; My engineering project

from scipy.spatial import distance
from imutils import face_utils
import RPi.GPIO as GPIO
import imutils
import dlib
import cv2
import threading
import time
import smtplib
from email.mime.multipart import MIMEMultipart
from email.mime.text import MIMEText
from email.mime.base import MIMEBase
from email import encoders
from email.mime.image import MIMEImage

fromaddr = "xxx@gmail.com"    # change the email address accordingly
toaddr = "yyy@gmail.com"
mail = MIMEMultipart()
 
mail['From'] = fromaddr
mail['To'] = toaddr
mail['Subject'] = "Attachment"
body = "Accident Happened"

VIB = 20
SMOKE = 16
BUZZER = 21
VIBRATOR = 26
MOTOR = 19
TRIG = 23                                  #Associate pin 23 to TRIG
ECHO = 24
#print ("Distance measurement in progress")
GPIO.setwarnings(False)
GPIO.setmode(GPIO.BCM)
GPIO.setup(BUZZER,GPIO.OUT)
GPIO.setup(VIB,GPIO.IN)
GPIO.setup(SMOKE,GPIO.IN)
GPIO.setup(SMOKE,GPIO.IN)
GPIO.setup(VIBRATOR,GPIO.OUT)
GPIO.setup(MOTOR,GPIO.OUT)
GPIO.setup(12,GPIO.OUT)
GPIO.setup(TRIG,GPIO.OUT)                  #Set pin as GPIO out
GPIO.setup(ECHO,GPIO.IN) 
p= GPIO.PWM(12,100)
p.start(0)
x=100

def ultrasonic():
  global x
  print ("Distance measurement in progress")
  p.ChangeDutyCycle(x)
  GPIO.output(TRIG, False)                 #Set TRIG as LOW
  #print "Waitng For Sensor To Settle"
  time.sleep(0.5)                            #Delay of 2 seconds

  GPIO.output(TRIG, True)                  #Set TRIG as HIGH
  time.sleep(0.00001)                      #Delay of 0.00001 seconds
  GPIO.output(TRIG, False)                 #Set TRIG as LOW

  while GPIO.input(ECHO)==0:               #Check whether the ECHO is LOW
    pulse_start = time.time()              #Saves the last known time of LOW pulse

  while GPIO.input(ECHO)==1:               #Check whether the ECHO is HIGH
    pulse_end = time.time()                #Saves the last known time of HIGH pulse 

  pulse_duration = pulse_end - pulse_start #Get pulse duration to a variable

  distance = pulse_duration * 17150        #Multiply pulse duration by 17150 to get distance
  distance = round(distance, 2)            #Round to two decimal points

  if distance > 2 and distance < 400:      #Check whether the distance is within range
    print ("Distance:",distance,"cm")  #Print distance with 0.5 cm calibration
  else:
    print ("Out Of Range")                   #display out of range
  if distance <=15:
         x=75
  if distance <=10:
         x=50
  if distance <=5:
         x=25
  if distance <5:
         x=0	

def Send_text_mail():
        #import smtplib
        #from email.mime.multipart import MIMEMultipart
        #from email.mime.text import MIMEText
	fromaddr = "xxx@gmail.com"
	toaddr = "yyy@gmail.com"
	msg = MIMEMultipart()
	msg['From'] = fromaddr
	msg['To'] = toaddr
	msg['Subject'] = "Driver Alert"
	body = "Your Driver Drunk\n Alcohal content detected"
	msg.attach(MIMEText(body, 'plain'))
	server = smtplib.SMTP('smtp.gmail.com', 587)
	server.starttls()
	server.login(fromaddr, "Mash@Asoft")
	text = msg.as_string()
	server.sendmail(fromaddr, toaddr, text)
	server.quit()

def sendMail():
    mail.attach(MIMEText(body, 'plain'))
    dat='opencv.png'
    attachment = open(dat, 'rb')
    image=MIMEImage(attachment.read())
    attachment.close()
    mail.attach(image)
    server = smtplib.SMTP('smtp.gmail.com', 587)
    server.starttls()
    server.login(fromaddr, "Mash@Asoft")
    text = mail.as_string()
    server.sendmail(fromaddr, toaddr, text)
    server.quit()

def capture_image():
    cv2.imwrite('opencv.png',frame)
    sendMail()


#def thread_function(name):
	#playsound.playsound('beep.mp3')
	


def eye_aspect_ratio(eye):
    A = distance.euclidean(eye[1], eye[5])
    B = distance.euclidean(eye[2], eye[4])
    C = distance.euclidean(eye[0], eye[3])
    ear = (A + B) / (2.0 * C)
    return ear


if __name__== "__main__":
	thresh = 0.25
	frame_check = 10
	detect = dlib.get_frontal_face_detector()
	#predict = dlib.shape_predictor("C:/Users/USER/Desktop/Drowsiness_darshan/shape_predictor_68_face_landmarks.dat")
	predict = dlib.shape_predictor("shape_predictor_68_face_landmarks.dat")

	(lStart, lEnd) = face_utils.FACIAL_LANDMARKS_IDXS["left_eye"]
	(rStart, rEnd) = face_utils.FACIAL_LANDMARKS_IDXS["right_eye"]
	#(nStart, nEnd) = face_utils.FACIAL_LANDMARKS_IDXS["nose"]
	cap=cv2.VideoCapture(0)
	flag=0
	GPIO.output(BUZZER,0)
	#ultrasonic()
	while True:
		ultrasonic()
		ret, frame=cap.read()
		frame = imutils.resize(frame, width=450)
		gray = cv2.cvtColor(frame, cv2.COLOR_BGR2GRAY)
		subjects = detect(gray, 0)
		if GPIO.input(SMOKE)==0:
                        print ("Alcohal Detected")
                        GPIO.output(BUZZER,1)
                        #GPIO.output(MOTOR,0)
                        x=0
                        p.ChangeDutyCycle(x)
                        time.sleep(2)
                        Send_text_mail()
		else:
                        print ("Alcohal Not Detected")
                        GPIO.output(BUZZER,0)
                        x=100
                        p.ChangeDutyCycle(x)
                        #GPIO.output(LIGHT,0)
		if GPIO.input(VIB)==0:
                        print ("Accident Detected")
                        #GPIO.output(MOTOR,0)
                        x=0
                        p.ChangeDutyCycle(x)
                        capture_image()
		else:
			GPIO.output(BUZZER,0)

		for subject in subjects:
			shape = predict(gray, subject)
			shape = face_utils.shape_to_np(shape)
			leftEye = shape[lStart:lEnd]
			rightEye = shape[rStart:rEnd]
			#nose=shape[nStart:nEnd]
			leftEAR = eye_aspect_ratio(leftEye)
			rightEAR = eye_aspect_ratio(rightEye)
			ear = (leftEAR + rightEAR) / 2.0
			leftEyeHull = cv2.convexHull(leftEye)
			rightEyeHull = cv2.convexHull(rightEye)
			#noseHull=cv2.convexHull(nose)
			cv2.drawContours(frame, [leftEyeHull], -1, (0, 255, 0), 1)
			cv2.drawContours(frame, [rightEyeHull], -1, (0, 255, 0), 1)
			#cv2.drawContours(frame, [noseHull], -1, (0, 255, 0), 1)
			if ear < thresh:
				flag += 1
				#print (flag)
				if flag >= frame_check:
					#x = threading.Thread(target=thread_function, args=(1,))
					#x.start()
					cv2.putText(frame, "****************ALERT!****************", (10, 30),cv2.FONT_HERSHEY_SIMPLEX, 0.7, (0, 0, 255), 2)
					cv2.putText(frame, "****************ALERT!****************", (10,325),cv2.FONT_HERSHEY_SIMPLEX, 0.7, (0, 0, 255), 2)
					#playsound.playsound('a.mp3')
					for i in range(4):
                                                GPIO.output(BUZZER,1)
                                                time.sleep(0.2)
                                                GPIO.output(BUZZER,0)
                                                time.sleep(0.2)
					x=0
					p.ChangeDutyCycle(x)

			else:
				flag = 0

		cv2.imshow("Frame", frame)
		key = cv2.waitKey(1) & 0xFF
		if key == ord("q"):
			cv2.destroyAllWindows()
			cap.release()
			break
