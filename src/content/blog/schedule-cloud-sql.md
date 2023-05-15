---
author: Nguyễn Quang Vinh
pubDatetime: 2023-05-15
title: "Schedule Cloud SQL instances to start and stop"
postSlug: schedule-cloud-sql
featured: true
draft: false
tags:
  - cloudsql
  - gcp
ogImage: ""
description: "Lập lịch để tự động khởi động và tắt Cloud SQL instance giúp tiết kiệm chi phí"
---

# 1. Chuẩn bị Cloud SQL instance

Trong guide này thì mình sẽ lập lịch cho instance Cloud SQL nên đầu tiên các bạn cần có một con Cloud SQL đang chạy. Với mình thì mình dùng luôn con jira-db đang chạy của mình.

# 2. Tạo Cloud Function để khởi động và tắt Cloud SQL instance

Vào phần [Cloud Function](https://console.cloud.google.com/functions) trên Cloud Console rồi chọn CREATE FUNCTION.

<p align="center">
  <img src="./image/Screenshot from 2023-05-15 15-01-42.png" alt="Create function">
</p>

Phần Eventarc Trigger ta chọn Cloud Pub/Sub, chọn topic đã có từ trước hoặc tạo mới, của mình thì tạo mới một topic lấy tên là InstanceMgmt.

<p align="center">
  <img src="./image/Screenshot from 2023-05-15 15-09-59.png" alt="Create trigger">
</p>

Sau khi xong thì Save Trigger rồi chon Next để sang bước tiếp theo. Phần này sẽ là code của function, mình chọn Go Runtime và xài đoạn code sẵn từ blog của Google như sau:

```Go
// Package p contains a Pub/Sub Cloud Function.
package p

import (
	"context"
	"encoding/json"
	"log"

	"golang.org/x/oauth2/google"
	sqladmin "google.golang.org/api/sqladmin/v1beta4"
)

// PubSubMessage is the payload of a Pub/Sub event.
// See the documentation for more details:
// https://cloud.google.com/pubsub/docs/reference/rest/v1/PubsubMessage
type PubSubMessage struct {
	Data []byte `json:"data"`
}

type MessagePayload struct {
	Instance string
	Project  string
	Action   string
}

// ProcessPubSub consumes and processes a Pub/Sub message.
func ProcessPubSub(ctx context.Context, m PubSubMessage) error {
	var psData MessagePayload
	err := json.Unmarshal(m.Data, &psData)
	if err != nil {
		log.Println(err)
	}
	log.Printf("Request received for Cloud SQL instance %s action: %s, %s", psData.Action, psData.Instance, psData.Project)

	// Create an http.Client that uses Application Default Credentials.
	hc, err := google.DefaultClient(ctx, sqladmin.CloudPlatformScope)
	if err != nil {
		return err
	}

	// Create the Google Cloud SQL service.
	service, err := sqladmin.New(hc)
	if err != nil {
		return err
	}

	// Get the requested start or stop Action.
	action := "UNDEFINED"
	switch psData.Action {
	case "start":
		action = "ALWAYS"
	case "stop":
		action = "NEVER"
	default:
		log.Fatal("No valid action provided.")
	}

	// See more examples at:
	// https://cloud.google.com/sql/docs/sqlserver/admin-api/rest/v1beta4/instances/patch
	rb := &sqladmin.DatabaseInstance{
		Settings: &sqladmin.Settings{
			ActivationPolicy: action,
		},
	}

	resp, err := service.Instances.Patch(psData.Project, psData.Instance, rb).Context(ctx).Do()
	if err != nil {
		log.Fatal(err)
	}
	log.Printf("%#v\n", resp)
	return nil
}
```

Điền thông tin Entry point của function là ProcessPubSub rồi chọn Deploy. Đợi vài phút cho function deploy xong.

# 3. Cấp quyền cho Cloud Function quản lý Cloud SQL instance

Truy cập vào mục [IAM](https://console.cloud.google.com/iam-admin) và thêm role cho "App Engine default service account" như trong hình:

<p align="center">
  <img src="./image/Screenshot from 2023-05-15 15-33-00.png" alt="Add role">
</p>

# 4. Tạo Cloud Scheduler Job để lập lịch cho Cloud Function

Vào phần [Cloud Scheduler](https://console.cloud.google.com/cloudscheduler) trên Cloud Console rồi chọn CREATE JOB.

<p align="center">
  <img src="./image/Screenshot from 2023-05-15 15-44-36.png" alt="Add job">
</p>

Điền các thông tin như hình, trong đó phần Frequency sẽ viết theo unix-cron giống như khi tạo cron job. Sau đó sang phần Configure the execution.

<p align="center">
  <img src="./image/Screenshot from 2023-05-15 18-07-16.png" alt="Add message body">
</p>

Chọn Target type là Pub/Sub rồi chọn topic lúc trước ta đã tạo là InstanceMgmt. Phần Message body thì sử dụng code json:

```json
{
  "Instance": "jira-db",
  "Project": "jira-vsi-lab",
  "Action": "start"
}
```

Trong đó Instance là tên của Cloud SQL instance, Project là tên của project chứa Cloud SQL instance, Action là start hoặc stop. Sau đó chọn Create.
Tạo xong job cho tự khởi động ta làm tương tự để tạo job cho tự tắt Cloud SQL instance. Như vậy là hoàn tất.
