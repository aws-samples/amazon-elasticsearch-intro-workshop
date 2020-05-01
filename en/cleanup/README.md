# Cleanup

This section explains how to delete resources created in this workshop. **Do not forget** to delete the resource after performing this workshop. Otherwise, **you will continue to be charged**.

## Amazon ES

1. Click [Services] in the top left of the AWS Management Console to display the list of services, and choose **[Elasticsearch Service]**.
2. Once displayed, choose **"workshop-esdomain"** in the list, and click **[Deleting a domain]** of **[Action]** button. When the pop-up menu is displayed, check the check-box, and click **[Delete]**.

## Firehose

1. Click [Services] in the top left of the AWS Management Console to display the list of services, and choose **[Kinesis]**.
2. Choose **"workshop-firehose"** in the Kinesis Firehose delivery stream, and click **[Delete delivery stream]** button. When the pup-up menu is displayed, click **[Delete delivery stream]** to confirm the deletion.
3. Delete the S3 bucket for storing the records that failed in inserting them into Amazon ES. Click [Services] in the top left of the AWS Management Console to display the list of services, and choose **[S3]**.
4. Check the check-box of **"workshop-firehose-backup-YYYYMMDD-YOURNAME"** from the S3 bucket list, and click **[Delete]** button n the menu at the top. After entering the bucket name in the pop-up menu, click **[Confirm]** to confirm the deletion.

## Kinesis Data Generator

1. Click [Services] in the top left of the AWS Management Console to display the list of services, and choose **[CloudFormation]**.
2. Change the region selection screen in the top right of the screen to **[Oregon]**.
3. Choose **"Kinesis-Data-Generator-Cognito-User"** from the list, click **[Delete]** button, and choose **[Deleting a stack]**.
4. Then, click [Services] in the top left of the AWS Management Console to display the list of services, and choose **[Cognito]**.
5. Click **[Manage User Pools]** to display the list, choose **[Kinesis Data-Generator Users]**, and delete the pool in the top right from **[Removing a pool]**.
6. Click **[Federated identity]** in the top left, and choose **[KinesisDataGeneratorUsers]**. Display the menu of [Deleting an ID pool] below, click **[Deleting an ID pool]** button, and then click **[Deleting a pool]** to confirm the deletion.

## Amazon SNS

1. Click [Services] in the top left of the AWS Management Console to display the list of services, and choose **[Simple Nortification Service]**.
2. Click **[Topics]** from the left menu, choose "**amazon_es_alert**", and click **[Delete]** button. When the pop-up menu is displayed, enter **"Delete this"**, and then confirm the **[deletion]**.
3. Click **[Subscriptions]** from the left menu, choose a  subscription for **"amazon_es_alert"** topic, and then click **[Delete]**.
4. At last, delete the IAM role used to send notifications to SNS topics from Amazon ES. Click [Services] in the top left of the AWS Management Console to display the list of services, and choose **[IAM]**.
5. Click **[Role]** from the left menu, enter **"amazones_sns_alert_role"** to display **"amazones_sns_alert_role"**, and then click **[Deleting a role]** button to delete.
6. In a similar manner, click **[Policies]** from the left menu, enter **"amazones_sns_alert_policy"** to display **"amazones_sns_alert_policy"**, and then click **[Delete]** from **[Policy action]** to to confirm the deletion.

This completes the cleanup.



