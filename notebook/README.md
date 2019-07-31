# Building a Movie Recommendation Engine using Amazon Personalize

In the previous section, you have setup an environment to use for the rest of the workshop. You also have deployed a pre-built, skeleton web application via a CloudFormation template.

From here, we will get into the meat of the workshop where you will be using 2 of the AWS managed Machine Learning serivce, Amazon SageMaker and Amazon Personalize.

## Workshop Roadmap (continued)

:warning: **This workshop works ONLY in N. Viginia (us-east-1)**  

- Overview
- Deploy the skeleton Web App
- [Launch Jupyter Notebook environment on Amazon SageMaker](#launch-jupyter-notebook-on-amazon-sagemaker)
- [Create components of Amazon Personalize for movie recommendation](#using-the-notebook-create-components-of-amazon-personalize-for-movie-recommendation), using the notebook.
- (Optional) Build Item-to-Item Recommendations using Amazon Personalize
- (Optional) Build Personal Ranking using Amazon Personalize
- [Setup Web App Configuration](#setup-web-app-configuration)
    - [Django Framework Config](#django-framework-config)
    - [Plug in Models](#plug-in-models)
    - (Optional) Additional Campaigns
- [Shutting Down](#shutting-down)
- [Conclusion](#conclusion)


## Launch Jupyter Notebook Environment on Amazon SageMaker

1. Click on **Amazon SageMaker** from the list of all services by entering Sagemaker into the **Find services** box. This will bring you to the Amazon SageMaker console homepage. 

1. To create a new Jupyter notebook instance, go to **Notebook instances** in the Amazon SageMaker console, and click the **Create notebook instance** button at the top of the browser window.

    ![sagemakerCreateNotebook](./images/sagemakerCreateNotebook.png)

1. Type *Personalize-Workshop* into the **Notebook instance name** text box, and then *ml.m5.2xlarge* into the **Notebook instance type**. Note, for this lab the majority of the work is performed by the Amazon Personalize service, so there is not need to launch a large, compute-optimized C5 or GPU-based instance type.

    ![sagemakerCreateNotebook2](./images/sagemakerCreateNotebook2.png)

    On the **Permission and encryption** section of the same page, choose **Create a new role** from the dropdown list. It will bring up the following popup window. Here, you can specify your S3 bucket or choose *Any S3 bucket* if you don't have S3 bucket created yet. For security purposes, we always recommend to choose *Specific S3 bucket* option but for the purpose of the lab, you could choose *Any S3 bucket*.  

    ![sagemakerCreateNotebook3](./images/sagemakerCreateNotebook3.png)

    Lastly on this wizard, scroll down to the **Git repositories** section and choose *Clone a public Git repository to this notebook instance only* option. Then enter the URL of this workshop as shown below. Click on **Create notebook instance** button. 

    https://github.com/rumiio/recommendation-using-Personalize   

    ![sagemakerCreateNotebook4](./images/sagemakerCreateNotebook4.png)

1. In another browser tab navigate to the IAM console homepage and find the IAM role SageMaker just created. You will be adding 2 policies: one is managed **AmazonPersonalizeFullAccess** and another is inline policy to give SageMaker permission to create a role from the notebook. 

    Click **Attach policies** button. 

    ![iamPersonalizeFullAccess](./images/iamPersonalizeFullAccess.png)

    Type *Personalize* in the search box. It will bring up PersonalizeFullAccess managed policy. Select the checkbox and then click on the **Attach policy** button. 

    ![iamPersonalizeFullAccess2](./images/iamPersonalizeFullAccess2.png)

    Look for the same IAM role again. This time, click on the **Add inline policy** link. 

    ![iamInlinePolicy](./images/iamInlinePolicy.png)

    On **Create policy** window, select JSON tab. Paste the following policy into the text area. Then click on **Review policy**.  

    ```
    {
        "Version": "2012-10-17",
        "Statement": [
            {
                "Sid": "IAMPolicy",
                "Effect": "Allow",
                "Action": [
                    "iam:CreateRole",
                    "iam:AttachRolePolicy",
                    "iam:PutRolePolicy"
                ],
                "Resource": "*"
            }
        ]
    }
    ```

    ![iamInlinePolicy2](./images/iamInlinePolicy2.png)

    Enter *Create_IAM_Role_Permission* as **Name**. Then click on the **Create policy** button.  

    ![iamInlinePolicy3](./images/iamInlinePolicy3.png)

    Repeat the previous steps to create another inline policy with the following permission. Replace *your-bucket-name* with actual bucket name. Name this plicy *S3PutBucketPolicy_Permission*. 

    ```
    {
        "Version": "2012-10-17",
        "Statement": [
            {
                "Sid": "S3Policy",
                "Effect": "Allow",
                "Action": "s3:PutBucketPolicy",
                "Resource": "arn:aws:s3:::your-bucket-name"
            }
        ]
    }
    ```


1. Wait until the notebook instance status is **InService**. This will take a few minutes once the creation process has started. Then click on **Open Jupyter**.

    ![sagemakerCreateNotebook5](./images/sagemakerCreateNotebook5.png)



## Using the Notebook, Create Components of Amazon Personalize for Movie Recommendation.  

1. A notebook consists of a number of cells; in SageMaker these will typically either be Code or Markdown cells. Markdown is used to allow for documentation to be defined inline with the code, giving the author a rich set of markdown formatting options. The first cell in this notebook, which is called **Get the Personalize boto3 Client**, is Markdown, and if you select any cell then the whole cell is highlighted.

    ![sagemakerNotebook1](./images/notebook1.png)

1. The first Markdown cell describes what the following Code cell is going to do – for the sake of this lab you do not have to understand the code that is being run in the Code cell, rather you should just appreciate what the notebook is doing and how you interact with a Jupyter notebook.

    ![sagemakerNotebook2](./images/notebook2.png)

1. To the left of a Code module is a set of empty braces **\[ \]**. By highlighting the cell and then selecting the Run command in the menu bar, the Jupyter notebook will execute this code, outputting and code outputs to the notebook screen and keeping any results data internally for re-use in future steps. Do this now to execute the first code cell.

    *Note: if a Markdown cell is highlighted, then clicking **Run** will move the highlight to the next cell*

1. While the code is executing the braces will change to be **\[*\]**, indicating that it is executing, and once complete will change to **\[1\]**. Future cells will have increasing numbers inside the braces, and this helps you see the order in which cells have been exected within the notebook. Directly below the code, but still within the Code cell, is the output from the code execution - this will include any error messages that your code has thrown.

    ![sagemakerNotebook3](./images/notebook3.png)

1. Now please continue to work through the notebook lab - read the comments prior to each Code cell in order to get an understanding as to what is going on, as these explain why we are doing each step and how it ties in to using the Amazon Personalize service.


# Setup Web App Configuration

## Django Framework Config

There are various components within the application that need some final configuration. The basics, such as the VPC, the Application Load Balancer, the Auto-Scaling Group and resultant EC2 images, are all good to go, but some configuration is necessary on the Django application framework that is hosting the application. This needs to be done by connecting into the instance using SSH.

There are many ways to connect to a remote machine using SSH. Feel free to follow the instructions from the prerequisites. This Lab Guide will continue with using SSH at the command line on an Apple Mac computer - your own method for establishing a connection may be different, but once connected the instructions are the same regardless of your platform combination.

1. In order to connect you need to have your downloaded key-pair from earlier in an accessible location. It also must not be publicly readable, so if you are on a Mac or Linux system you can fix this with the following command, remembering to replace **myPersonalizeWorkshopKey.pem** with your key name!

    ```
    chmod 400 myPersonalizeWorkshop.pem
    ```
 1. Go to the the EC2 console page, go to the **Instances** menu on the left, and find your one running instance. Select it, and make a note of (or copy) the **IPv4 Public IP** for your instance.

     ![selectEC2details](./images/selectEC2details.png)

1. Go to your computer CLI, and navigate to the directory containing your key-pair. Issue the following command to connect via SSH, changing the key-pair filename and IP address as necessary, and you should see results similar to what follows. You will see a warning about the authenticity of the host then just enter **yes** at the prompt.

     ![sshLogin](./images/sshLogin.png)

1. Navigate to the root of the Django project, and configure the single-line run script to use the private IP address of the EC2 instance. This is on the previous EC2 details screen just underneath the Public IP - you will need this again, so keep it handy. Simply replace the IP address with yours, keeping the trailing **:8000** - my server’s private IP is 10.192.11.223.

    ```
    $ cd personalize-video-recs/videorecs/
    $ nano runmyserver
    --- {editor screen} ---
    python manage.py runserver 10.192.11.223:8000
    ```

1. Django only allows access via pre-defined source IP addresses. Naturally, these could be open to the internet, but they recommend only exposing it the instance private IP address (for internal calls) and to your front-end load balancer. You already have a reference to the private IP address, so you now need to extract the Load Balancer DNS entry. Go back to the EC2 console screen, but this time select **Load Balancers** on the left-hand menu; select your Application Load Balancer and in the details screen that comes up select the DNS name and store it for later. 

     ![loadbalancerDNS](./images/loadbalancerDNS.png)

1. While we’re collecting data, move to the **Amazon RDS** service section of the console, select **Databases** from the left-hand menu and select the Lab database called **summitpersonalizelab** from the list. In the details screen copy the DNS endpoint for the database and store it for later.

     ![rdsDNS](./images/rdsDNS.png)

1. Go back to your SSH session window. You now need to edit two entries in the file - one is called **ALLOWED_HOSTS** and the other is **HOST** entry in the **DATABASES** section. Edit the file, then in the editor window find the two relevant lines and edit them so that they look like that shown below, but with your IP and DNS entries

    ```
    $ nano videorecs/settings.py
    --- {ALLOWED_HOSTS line - server private IP and ALB DNS} ---

    ALLOWED_HOSTS = ['10.192.11.151', 'TestS-Appli-ADS60FMCKPMG-1862985075.us-east-1.elb.amazonaws.com']

    --- {DATABASES HOSTS line - RDS DNS} ---

        'HOST': 'summitpersonalizelab.c0azewoaia5d.us-east-1.rds.amazonaws.com',
    ```

1. Finally, the RDS database is postgres, and we have included the **pgcli** tool with this deployment. If you wish to use it then you need to edit the startup script for the utility to point to the RDS DNS entry. You also need to know the password, which you may have noticed in the **settings.py** file, and it’s **recPassw0rd**

    ```
    $ nano pgcli
    --- {editor screen} ---
    /home/ec2-user/.local/bin/pgcli -h summitpersonalizelab.c0azewoaia5d.us-east-1.rds.amazonaws.com -u vidrecdemo -d videorec
    ```

1. You are now ready to run the application server! Simply execute the **runmyserver** script, and you should see status messages appearing quickly - these initial ones are the Load Balancer health-checks, and after a minute or so the instance should be declared healthy by the Load Balancer Target Group. Note, you will see some warnings around the psycopg2 component, but this can be ignored.

    ```
    $ ./runmyserver

    System check identified no issues (0 silenced).
    May 06, 2019 - 14:53:03
    Django version 1.11.18, using settings 'videorecs.settings'
    Starting development server at http://10.192.11.223:8000/
    Quit the server with CONTROL-C.
    [06/May/2019 14:53:14] "GET /recommend/ HTTP/1.1" 200 2893
    [06/May/2019 14:53:32] "GET /recommend/ HTTP/1.1" 200 2893
    [06/May/2019 14:53:44] "GET /recommend/ HTTP/1.1" 200 2893
    ```

1. The URL of the server is your ALB followed by the ‘/recommend/’ path, although there is also an ‘/admin/’ path that we’ll use later. For now connect to your server - in my example the server can be found at 

    http://perso-appli-1ig76667ghapa-70965737.us-east-1.elb.amazonaws.com/recommend/

1. You should see the following screen in your browser - no Model Precision Metrics are available, as we haven’t added any models yet to the application. You can also see that documentation for this is present, but be aware that it may not be 100% up to date with coding changes on the demo.

     ![appFrontScreen](./images/appFrontScreen.png)

1. If you hit **Select Random User** then you’ll be taken to the main Recommendation screen, which starts by showing you a random user’s top-25 movie review titles. However, you’ll see on the Model dropdown on the left that there are no models available, and if you change the Personalize Mode to either Personal Ranking or Similar Items then it’s the same story - you can see the movie reviews, and most-popular titles in a genre, but no recommendations. We need to get the solutions and campaigns built in the notebook, then you can come back and plug in the models.

     ![appRecNoModels](./images/appRecNoModels.png)


## Plug in Models

The application uses the Django Administration feature to define models that are available to the application. This allows multiple models of different types to be configured, and injected or removed from the application at any time. There are three modes of operation of the application:

- **Recommendations** - standard recommendations, allowing different 2 models to be compared at once
- (Optional) **Personal Ranking** - re-ranks popular films in a genre, with a single model on-screen at once
- (Optional) **Similar Items** - shows items similar to others, with a single model on-screen at once. You can optionally send this list through a Personal Ranking model if you have one definedA

Each of these modes allows multiple models of their type to be used, but each mode can only show one or two different models simultaneously - however, you can choose any configured model at any time.

1. By default the admin user within Django does not exist - you need to create one. In your SSH session, ensure that the web server is running again, and then create a second SSH session. In that session, create the first superuser using the following commands. Please note down your username and password, as no-one will be able to retrieve it for you; in the Lab we suggest using **admin** and **DoNotH@ckMe** accordingly, but use any email address you like. You can then close the second SSH session

    ```
    cd ~/personalize-video-recs/videorecs/
    python manage.py createsuperuser

    Username (leave blank to use 'ec2-user'): admin
    Email address: anyone@email.com
    Password:
    Password (again):
    Superuser created successfully.
    ```

1. Login to the Django Administration site. This is at the same URL as the main application, but replace **/recommend** with **/admin** at the end of the URL. This will bring up the following screen, so login now with the credentials that you just created

     ![djangoAdmin](./images/djangoAdmin.png)

1. This brings up the Site Administration screen, which show entries for Groups and Users (which we don’t need), but also a section called **Recommend** where you can add **Personalize models** to the app. Click on **+Add** link to begin to add a new model

1. Back on the AWS Console, go to the **Amazon Personalize** service console, select the **summit-recs-dataset-group** and then on the left-hand menu click **Campaigns**. This will show your **summit-lab-recs-campaign**, but if you created the two additional solutions earlier then at this point you could could first go into the **Solutions and recipes** menu item, click on each of these two additional solutions and create the associated campaigns quickly via the console (or feel free to skip this). Once you’re back on the **Campaigns** menu you should see this

     ![campaignList](./images/campaignList.png)

1. Click on the **sf-loft-lab-recs-campaign** and you’ll see the **Campaign ARN** - copy this, and head back to the admin screen. Enter **Personal Recommendations** for the model name, enter the ARN where it asks, ensure that the **Model type** is set for recommendations and set the **Model sort order** to 1. Click on **SAVE** to save the definition.

     ![djangoAddModel](./images/djangoAddModel.png)

1. The application will use the sort order field to decide how to order models in the on-screen drop-downs. Only models of the right type are shown on the relevant screen, but there is no validation that you have entered the correct model type, and if you put a SIMS model on the Rankings screen then the application will throw errors.

1. If you also have a SIMS or Personal Ranking campaign then go ahead and add them now in the same way. You can then close the admin screen and head back to the main application web page

1. The main screen now shows the three models (or maybe just one) that we’ve built - it lists the precision metrics for each one, and as you add or remove models from the Django Administration page the changes will be reflected here. Now click on the **Select Random User** button

     ![appFrontScreenWithModels](./images/appFrontScreenWithModels.png)

1. The screen will look as before, but now if you click on the **Model 1** drop-down you will see that our one Recommendation model is present - if you select it then the screen will refresh to show recommendations for this user using that model.

     ![appRecWithModels](./images/appRecWithModels.png)

1. You can step through users to see how these look for different demographics of users. If you had multiple Recommendation models defined then they would also be in the two model drop-downs, and you’d be able to show two completely different recommendation models, based upon different user demographic or item metadata, allowing you to compare and contrast different approaches.

1. (Optional) Try out the **Personal Ranking** personalize mode - this takes a list of the most popular movies in the dataset, either as a whole or in just a single genre. This will take that list and re-rank it into an order for this particular user, ensuring that the ones that are most likely to be interested in are shown first.

1. (Optional) Finally, try the **Similar Items** personalize mode - this starts with the user’s top-ranked film, and finds a list of films that people who watched this also watched. This is done without reference to the user’s preferences, and the list is generated based upon what’s in the dataset as a whole. However, if you also have a **Personal Ranking** model defined then the Ordering drop-down will re-rank this list into one that is more suited to the user’s preferences.



# Shutting Down

## Deleting each compoments of Amazon Personalize

1. Open the notebook called **personalize_cleanup.ipynb**. 
1. Enter your account number in the first cell. 
1. Run all cells to delete each components of Amazon Personalize. 

## Shutting Down the SageMaker Notebook Instance

1. Open the Amazon SageMaker console and click on **Notebook instances**

1. Find the notebook instance listed as *Personalize-Workshop*, select its radio button and then click the **Actions** dropdown.

     ![terminateNotebook](./images/terminateNotebook.png)

1. Click **Stop** to stop the Notebook Instance. This does not delete the underlying data and resources. After a few minutes the instance status will change to *Stopped*, and you can now click on the **Actions** dropdown again, but this time select **Delete**.


## Shutting Down the CloudFormation Stack

The CloudFormation stack we deployed in this step contains multiple resources that we’ll to terminate:

- EC2 Instance
- Application Load Balancer
- Networking: Internet Gateway, VPC, Subnets, NAT Gateways, Route Table

To delete this:

1. On the **Stacks** page in the CloudFormation console, select the stack that you want to delete. The stack must be currently running.
1. In the stack details pane, choose **Delete**.
1. Select **Delete stack** when prompted.

Note: After stack deletion has begun, you cannot abort it. The stack proceeds to the *DELETE_IN_PROGRESS* state.


# Conclusion

Upon completion of this lab you will have performed the following:

- Launched a Jupyter notebook from with the Amazon SageMaker service
- Imported external files into the notebook environment
- Seen how to enable Preview services within a notebook (assuming your account has been whitelisted for Preview access)
- Used the pandas libraries to do some pre-processing of the source data
- Built and deployed an ML model based upon the HRNN algorithm
- Tested your model via just a few lines of code
- Deployed your model into a live application

You should now be able to embed this model from within your own application code, using any language that is supported by the AWS SDK. Happy recommending!
