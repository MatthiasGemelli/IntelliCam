
# AWS Lambda Script
by Matthias Gemelli, October 2017
- Tested with AWS Lambda, Python 2.7
- Code will NOT run standalone, as it uses IAM Role associated to Lambda to get S3/Reko access
- to run locally, check my other scripts (which pass AWS credentials...)
- just cut & paste into AWS Lambda (make sure you choose Python 2.7 as environment)

```python
#Intelli_Cam rev3 - Matthias Gemelli

from __future__ import print_function
import boto3
from decimal import Decimal
import json
import urllib
from random import random
import base64
from s3transfer.manager import TransferManager
import datetime
import time
import csv

# 1st define the different input/output paths/names
intellicam_bucket    = 'xxx.xxx.xxx'
intellicam_alarmlist = 'reko_alarms.csv'   # Label / Category / Alarm
intellicam_csv       = 'logs/reko_labels_'
intellicam_log       = 'logs/reko_logs_'        # logging to this file

# 2nd - establish S3 and REKOGNITION session, with IAM role (Lambda execute, S3+REKO access)
client_reko = boto3.client('rekognition')
client_s3 = boto3.client('s3')

# 3rd - function to retrieve my alarm list from S3 instead of hard-coding
#format= label/category/Alarm like 'Suit,Clothing,Alarm' or 'Female,People,Alarm'
def get_alarms(bucket, key):
    ##client_s3.download_fileobj(my_bucket, my_s3file, data)  #have to declare file object first?
    filepath = '/tmp/alarm.txt'
    alarmlist = []
    client_s3.download_file(bucket, key, filepath)
    with open(filepath) as fp:  
        for cnt, line in enumerate(fp):
            #print (line.split(',')[0] + "-" + line.split(',')[2])
            if line.split(',')[2].strip() == 'Alarm': 
                alarmlist.append(line.split(',')[0].strip())
    return alarmlist   #as list/array, not dict!

# 4th - function to call REKOGNITION API
def detect_labels(bucket, key):
    response = client_reko.detect_labels(
        Image={"S3Object": {"Bucket": bucket, "Name": key}},
        MaxLabels=50,
        MinConfidence=30)
    return response

# 5th - function to write txt to S3 file
def write_txt(txt_input, s3bucket, s3target):
    file = open('/tmp/tempcsvfile.csv', 'w')  #write to temp file
    file.write(txt_input)
    file.close()
    #now write file to S3
    file = open('/tmp/tempcsvfile.csv', 'r')
    client_s3.upload_fileobj(file, Bucket=s3bucket, Key=s3target)
    file.close()
    return 'File written'

# --------------- Main handler ------------------
def lambda_handler(event, context):
    now = datetime.datetime.now()
    print("------IntelliCam-" + str(now.year) + str(now.month) + str(now.day))
    # Get the S3 object from the event
    s3_bucket   = event['Records'][0]['s3']['bucket']['name']
    s3_key      = urllib.unquote_plus(event['Records'][0]['s3']['object']['key'].encode('utf8'))
    event_time  = event['Records'][0]['eventTime']
    event_srcip = event['Records'][0]['requestParameters']['sourceIPAddress'] 
    event_regio = event['Records'][0]['awsRegion']
    event_user  = event['Records'][0]['userIdentity']
    # compile text for LOG file
    log_txt     = "---New Lambda Event--\n"
    log_txt     += "Event time: " + str(event_time) + "\n"
    log_txt     += "Source IP : " + str(event_srcip) + "\n"
    log_txt     += "Region    : " + str(event_regio) + "\n"
    log_txt     += "User      : " + str(event_user) + "\n"
    log_txt     += "S3 Bucket : " + str(s3_bucket) + "\n"
    log_txt     += "S3 Key    : " + str(s3_key) + "\n"
    log_txt     += "Lambda Event:\n" + str(event) + "\n"

    #now get my alarmlist  - e.g. when to trigger alarm or not
    alarms = get_alarms(intellicam_bucket,intellicam_alarmlist)

    #now call REKOGNITION
    try:
        reko_alarm = False   #by default, no alarm
        reko_csv   = ""      #collect the labels to CSV
        reko_text = detect_labels(s3_bucket, s3_key)  #call REKOGNITION
        reko_labl = reko_text['Labels']   #get the labels
        labels_alarm    = s3_key + ' alarm for labels:'  #text for logging
        labels_no_alarm = s3_key + ' no alarm for labels:' #txt for logging

        for label in reko_labl:    # now iterate through each label
            label_name = str(label['Name'])
            label_conf = str(label['Confidence'])
            reko_csv   += '\n' + s3_key+','+str(label['Name'])+','+str(round(label['Confidence']))
            if label_name in alarms:    #now check for alarm 
                reko_alarm = True       # my alarm trigger
                labels_alarm += label_name + ','   #and collect all labels
            else: labels_no_alarm += label_name + ','
        print(labels_alarm) # check this in AWS Cloudwatch / Logs
        print(labels_no_alarm) #check this output in AWS cloudwatch / logs
        
        #now move the file accordingly to right folder alarmyes/no
        if reko_alarm:        #copy file to Alarm_Yes folder
            copy_source = {'Bucket': s3_bucket,'Key': s3_key}
            out = s3_key.replace('upload/','jpg_alarm/')   #
            client_s3.copy(copy_source, s3_bucket, out)
            client_s3.delete_object(Bucket=s3_bucket, Key=s3_key)  # delete from upload
            print('write JPG to ' + out)
        else:       #no Alarm
            copy_source = {'Bucket': s3_bucket,'Key': s3_key}
            out = s3_key.replace('upload/','jpg_quiet/')
            client_s3.copy(copy_source, s3_bucket, out)
            client_s3.delete_object(Bucket=s3_bucket, Key=s3_key)
            print('write JPG to ' + out)

        #finally write label.csv
        timestr  = str(now.year) + str(now.month) + str(now.day) 
        timestr += '_' + str(now.hour) + str(now.minute) + '_' + str(now.second)+str(now.microsecond)
        #write_txt(log_txt,  s3_bucket, intellicam_log + timestr + '.log')  #detailed log for diagnosing
        if reko_alarm: 
            out = s3_key.replace('upload/','csv_alarm/') + timestr + '.csv'
            write_txt(reko_csv, s3_bucket,  out)
            print ('write CSV to ' + out)
        else:
            out = s3_key.replace('upload/','csv_quiet/') + timestr + '.csv'
            write_txt(reko_csv, s3_bucket,  out)
            print ('write CSV to ' + out)
        
        print("-----Intellicam done processing " + s3_key)
        
    except Exception as e:
        print('Matthias Error')
        print(e)
        raise e
# END of the story
```
