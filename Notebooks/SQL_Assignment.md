# Investigating a Drop in User Engagement
from Mode https://mode.com/sql-tutorial/a-drop-in-user-engagement/

## The Problem
User engagement has dropped off, and we need to determine why.

A number of hypotheses have been presented. We will try to determine if the drop is coming from: fewer users signing up, new vs old users, users by device type, or from customer engagement emails (by type).

![image](https://user-images.githubusercontent.com/46611104/143505231-7ca8ca06-fe56-4822-b516-5755305ad616.png)

## The Investigation

### Review the Users Table and Compare All Users to New Activations
```SQL
SELECT DATE_TRUNC('day', created_at) AS day,
COUNT(*) AS all_users,
COUNT(CASE WHEN activated_at IS NOT NULL THEN user_id ELSE NULL END) AS activated_users
FROM tutorial.yammer_users
WHERE created_at BETWEEN '2014-06-01' AND '2014-09-01'
GROUP BY 1
ORDER BY 1
```
![image](https://user-images.githubusercontent.com/46611104/143505377-1c098b7a-232d-42b0-a889-3868bbdc4bcb.png)

We can see a slight drop in the first week of August in terms of new user activations, but it returns to normal thereafter. This doesn't appear to be the cause.

### Review the Combined Users and Events Table to see if the Drop Correlates with User Age on Platform
```SQL
SELECT DATE_TRUNC('week', z.occurred_at) AS week,
COUNT(DISTINCT user_id) AS total_users,
COUNT(DISTINCT CASE WHEN z.user_age >= 70 THEN z.user_id ELSE NULL END) AS "10+ weeks",
COUNT(DISTINCT CASE WHEN z.user_age < 70 AND z.user_age >= 63 THEN z.user_id ELSE NULL END) AS "9 weeks",
COUNT(DISTINCT CASE WHEN z.user_age < 63 AND z.user_age >= 56 THEN z.user_id ELSE NULL END) AS "8 weeks",
COUNT(DISTINCT CASE WHEN z.user_age < 56 AND z.user_age >= 49 THEN z.user_id ELSE NULL END) AS "7 weeks",
COUNT(DISTINCT CASE WHEN z.user_age < 49 AND z.user_age >= 42 THEN z.user_id ELSE NULL END) AS "6 weeks",
COUNT(DISTINCT CASE WHEN z.user_age < 42 AND z.user_age >= 35 THEN z.user_id ELSE NULL END) AS "5 weeks",
COUNT(DISTINCT CASE WHEN z.user_age < 35 AND z.user_age >= 28 THEN z.user_id ELSE NULL END) AS "4 weeks",
COUNT(DISTINCT CASE WHEN z.user_age < 28 AND z.user_age >= 21 THEN z.user_id ELSE NULL END) AS "3 weeks",
COUNT(DISTINCT CASE WHEN z.user_age < 21 AND z.user_age >= 14 THEN z.user_id ELSE NULL END) AS "2 weeks",
COUNT(DISTINCT CASE WHEN z.user_age < 14 AND z.user_age >= 7 THEN z.user_id ELSE NULL END) AS "1 week",
COUNT(DISTINCT CASE WHEN z.user_age < 7 THEN z.user_id ELSE NULL END) AS "Less than a week"
FROM
(
SELECT u.user_id,
e.occurred_at,
EXTRACT('day' FROM '2014-09-01'::TIMESTAMP - u.activated_at) AS user_age
FROM tutorial.yammer_users u
JOIN tutorial.yammer_events e
ON u.user_id = e.user_id
AND e.event_type = 'engagement'
AND e.event_name = 'login'
AND e.occurred_at BETWEEN '2014-05-01' AND '2014-09-01'
WHERE u.activated_at IS NOT NULL
) z
GROUP BY 1
```
![image](https://user-images.githubusercontent.com/46611104/143505544-fe60bd7c-a407-4d43-81eb-0e32bfe8e441.png)

It appears that the 10+ week cohort drop-off is responsible for the low engagement.

### Review the Events Table to Determine if There is a Difference Based on Device Used
```SQL
SELECT DATE_TRUNC('week', occurred_at) AS week,
COUNT(DISTINCT user_id) AS total_users,
COUNT(DISTINCT CASE WHEN device IN ('macbook pro','lenovo thinkpad','macbook air','dell inspiron notebook',
          'asus chromebook','dell inspiron desktop','acer aspire notebook','hp pavilion desktop','acer aspire desktop','mac mini') 
          THEN user_id ELSE NULL END) AS desktop,
COUNT(DISTINCT CASE WHEN device IN ('ipad air','nexus 7','ipad mini','nexus 10','kindle fire','windows surface',
        'samsumg galaxy tablet') THEN user_id ELSE NULL END) AS tablet,
COUNT(DISTINCT CASE WHEN device IN ('iphone 5','samsung galaxy s4','nexus 5','iphone 5s','iphone 4s','nokia lumia 635',
       'htc one','samsung galaxy note','amazon fire phone') THEN user_id ELSE NULL END) AS mobile
FROM  tutorial.yammer_events
WHERE event_type = 'engagement'
AND event_name = 'login'
AND occurred_at BETWEEN '2014-05-01' AND '2014-09-01'
GROUP BY 1
ORDER BY 1
```
![image](https://user-images.githubusercontent.com/46611104/143505663-4db770c0-360c-48fc-9cbc-1012999f0999.png)

We see a drop in mobile and tablet users, perhaps there is an issue with the mobile app?

### Review the Emails Table to See if the Engagement Emails are Affecting Engagement
```SQL
SELECT DATE_TRUNC('week', occurred_at) AS week,
COUNT(CASE WHEN action = 'sent_weekly_digest' THEN user_id ELSE NULL END) AS weekly_digest_emails_sent,
COUNT(CASE WHEN action = 'sent_reengagement_email' THEN user_id ELSE NULL END) AS reengagement_emails_sent,
COUNT(CASE WHEN action = 'email_open' THEN user_id ELSE NULL END) AS email_opens,
COUNT(CASE WHEN action = 'email_clickthrough' THEN user_id ELSE NULL END) AS email_clickthroughs
FROM tutorial.yammer_emails 
WHERE occurred_at BETWEEN '2014-05-01' AND '2014-09-01'
GROUP BY 1
ORDER BY 1
```
![image](https://user-images.githubusercontent.com/46611104/143505750-92d7f5a5-8592-40ac-aefe-8bc8ad4a9ff1.png)

There is a significant drop in email click-throughs that coincides with the drop in engagement, but is it due to the weekly digest email or the re-engagemetn email?

### Review the Emails Table to Determine Open and Click-Through Rates for Each Type of Email
```SQL
SELECT week,
weekly_opens/CASE WHEN weekly_emails = 0 THEN 1 ELSE weekly_emails END::FLOAT AS weekly_open_rate,
weekly_ctr/CASE WHEN weekly_opens = 0 THEN 1 ELSE weekly_opens END::FLOAT AS weekly_ctr,
retain_opens/CASE WHEN retain_emails = 0 THEN 1 ELSE retain_emails END::FLOAT AS retain_open_rate,
retain_ctr/CASE WHEN retain_opens = 0 THEN 1 ELSE retain_opens END::FLOAT AS retain_ctr
FROM
(
SELECT DATE_TRUNC('week', e1.occurred_at) AS week,
COUNT(CASE WHEN e1.action = 'sent_weekly_digest' THEN e1.user_id ELSE NULL END) AS weekly_emails,
COUNT(CASE WHEN e1.action = 'sent_weekly_digest' THEN e2.user_id ELSE NULL END) AS weekly_opens,
COUNT(CASE WHEN e1.action = 'sent_weekly_digest' THEN e3.user_id ELSE NULL END) AS weekly_ctr,
COUNT(CASE WHEN e1.action = 'sent_reengagement_email' THEN e1.user_id ELSE NULL END) AS retain_emails,
COUNT(CASE WHEN e1.action = 'sent_reengagement_email' THEN e2.user_id ELSE NULL END) AS retain_opens,
COUNT(CASE WHEN e1.action = 'sent_reengagement_email' THEN e3.user_id ELSE NULL END) AS retain_ctr
FROM tutorial.yammer_emails e1
LEFT JOIN tutorial.yammer_emails e2
ON e2.occurred_at >= e1.occurred_at
AND e2.occurred_at < e1.occurred_at + INTERVAL '5 MINUTE'
AND e2.action = 'email_open'
AND e2.user_id = e1.user_id
LEFT JOIN tutorial.yammer_emails e3
ON e3.occurred_at >= e2.occurred_at
AND e3.occurred_at < e2.occurred_at + INTERVAL '5 MINUTE'
AND e3.action = 'email_clickthrough'
AND e3.user_id = e2.user_id
WHERE e1.action IN('sent_weekly_digest', 'sent_reengagement_email')
AND e1.occurred_at BETWEEN '2014-06-01' AND '2014-09-01'
GROUP BY 1
) a
ORDER BY 1
```
![image](https://user-images.githubusercontent.com/46611104/143505878-8c7b5e13-f5d1-4705-8ed4-c8fe4d81cb09.png)

It appears as though the Weekly Digest email is not driving engagement with the platform as measured by click-through rate. Time to present our findings!
