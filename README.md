# IntelliCam
Security Camera with Artifical Intelligence processing (Raspberry PI -> AWS Rekognition)
Copyright Matthias Gemelli, Oct 2017


## 0 - IntelliCam Features
- automatic image recording & upload (Motion triggers s3_upload.py)
- AWS S3 bucket triggers AWS Lambda on each upload (for *.jpg files in upload/ folder)
- AWS Lambda processes each image with AWS Rekognition and:
- reads alarm list (Person, Car, ...) from S3 (reko_alarms.csv)
- calls AWS Rekognition for the uploaded image, compares image labels against alarm list
- when alarm triggered, moves image to folder jpg_alarm/, otherwise to jpg_quiet/
- saves all recorded image labels to folder csv_alarm/, otherwise to csv_quiet/

### Planned Improvements & Features:
- analysis of alarms (over time, labels, false negatives...)
- analysis of AWS cost (is this solution affordable?)
- better code structure (config files, s3 folder structure...)
- notification routine (e.g. S3 Lambda triggered by jpg_alarm/ file)
- Image processing (Thumbnails, Daily Gallery)
- email notifications (e.g. AWS SES)


## 1 - Background & Initial Challenge:
Setting up a Raspberry-PI-powered security camera is quite easy - thanks to software packages like [Motion] <https://motion-project.github.io/> or [MotionEyeOS] <https://github.com/ccrisan/motioneyeos>.

**The challenge?**  After setting up the Raspi camera I recorded 300..400 images. **Per Day.**   
Way too many pictures: my car turning into the driveway, neighbors cats, etc...  But I only wanted to see who is entering my property! I finally switched off the camera. 

Until I stumbled upon...


## 2 - The Solution: "AI" with Amazon REKOGNITION
I read the excellent blog posts by Mark West and realized that I could simply filter the images with Amazon's "Artificial Intelligence" service **AWS REKOGNITION**.
Mark explains how he let AWS Rekognition automatically detect (and notify him) if there were people in the picture. Exactly what I was looking for! 

- [Mark West: Smarten up your Pi Zero Web Camera with Image Analysis and AWS, pt1]  
<https://www.bouvet.no/bouvet-deler/utbrudd/smarten-up-your-pi-zero-web-camera-with-image-analysis-and-amazon-web-services-part-1>
- [Mark West: Smarten up your Pi Zero Web Camera with Image Analysis and AWS, pt2]  
<https://www.bouvet.no/bouvet-deler/utbrudd/smarten-up-your-pi-zero-web-camera-with-image-analysis-and-amazon-web-services-part-2>

Also stumbled over the post by Amit Agrawal on the AWS blog, providing me with a simple and illustrated tutorial:   
[Amit Agrawal: Capture and Analyze Customer Demographic Data Using Rekognition & Athena]
<https://aws.amazon.com/de/blogs/ai/capture-and-analyze-customer-demographic-data-using-amazon-rekognition-amazon-athena/>


## 3 - My Implementation (RPi --> S3 --> Lambda)
If you want to follow my implementation, I suggest the following steps:

1. Try the AWS Rekognition Demo with your sample images
2. Setup AWS S3, IAM Role and AWS Lambda - test by manually uploading some images. 
Use this [code for Lambda]<https://github.com/MatthiasGemelli/IntelliCam/blob/master/IntelliCam_lambda_py.md>
3. Setup RPi with Motion and automatic image upload to S3
4. Image download

### 3.1 - Try the AWS Rekognition Demo
Ideally you have some sample images from your Raspberry camera, upload them to the AWS Rekognition Demo at <https://eu-west-1.console.aws.amazon.com/rekognition/>.
This should give you an understanding what labels to expect (People, Person, Automobile, Building...).


### 3.2 - Setup AWS IAM role, AWS Lambda, AWS S3 bucket with trigger
Following Amit Agrawals post you should:

- create a new AWS IAM role with Lambda Execution and full access to S3 and Rekognition
- create a new AWS Lambda routine using my code 
- create a new AWS S3 bucket with trigger linked to the new Lambda routine
- upload some image to your S3 upload folder
- check Cloudwatch for log messages and troubleshooting
- check the processed images and labels

### 3.3 - Setup RPi with Motion and automatic S3 image upload
Using my S3 upload scripts...


### 3.4


## 4 - Lessons Learned, Next Steps

My Conclusions:

- Shit. It works! 



- Reddit AWS post <https://www.reddit.com/r/aws/comments/759ovz/aws_rekognition_fail/>
- Stack Overflow post <https://stackoverflow.com/questions/46649946/aws-rekognition-fails-to-detect-people-in-4-5>
