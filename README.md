## 目的：在使用Git時往往都是使用ssh或者http上傳或下載，如果要做到縝密的資安的話可以結合MFA的驗證，透過驗證更改Access Key ID、ACCESS_KEY以及ACCESS_TOKEN等參數進一步加強資安管控。
此篇就是在介紹如何使用MFA並結合git使用的簡單介紹，相關延伸也可以延伸至S3的透過MFA的管控。
在文章中會分幾個部分，操作程序、故障排除、參考資料、以及IAM policy

## 最後更新日期：2023.07.28

## 操作程序

1.開通IAM user、啟動MFA、開啟Access Keys、並且視需求開啟SSH public keys for AWS CodeCommit或者HTTPS Git credentials for AWS CodeCommit。本文章將以HTTPS Git credentials for AWS CodeCommit作為示範。

2.透過AWS configure登入使用者，如果要更改可以至~.aws/configure以及~.aws/crenditial進行設定。

3.透過aws sts 指令取得access token等資訊
```
aws sts get-session-token --serial-number arn:aws:iam::123456789012:mfa/example --token-code 123456
#其中arn:aws:iam::123456789012:mfa/example 為Multi-factor authentication (MFA)的設定mfa名稱視設定時的給定而定可能會跟iam user名稱不同，這邊需要注意一下
```
取得的資訊格式如下：

```
{
    "Credentials": {
        "SecretAccessKey": "secret-access-key",
        "SessionToken": "temporary-session-token",
        "Expiration": "expiration-date-time",
        "AccessKeyId": "access-key-id"
    }
}

```
4.將上述資料依序輸入區域變數中

Linux
```
export AWS_ACCESS_KEY_ID=example-access-key-as-in-previous-output
export AWS_SECRET_ACCESS_KEY=example-secret-access-key-as-in-previous-output
export AWS_SESSION_TOKEN=example-session-token-as-in-previous-output

```
在設定之後可以透過以下指令查看是否有設定成功
```
echo $AWS_ACCESS_KEY_ID
echo $AWS_SECRET_ACCESS_KEY
echo $AWS_SESSION_TOKEN
```
若要刪除可以採取以下步驟 environment variables

```
unset AWS_ACCESS_KEY_ID
unset AWS_SECRET_ACCESS_KEY
unset AWS_SESSION_TOKEN
```

5.在要clone或者pull push的Codecommit中點選Clone URL，並選取Clone HTTPS(GRC)，以git clone搭配複製的網址clone資料，例如：
```
git clone https://git-codecommit.ap-northeast-1.amazonaws.com/v1/repos/example
```

## 權限整理(待整理)
get-session token



## 錯誤訊息整理：
[1]無法存取 'https://git-codecommit.ap-northeast-1.amazonaws.com/v1/repos/Bing-test/'：The requested URL returned error: 403

MAC:刪除鑰匙圈存取

[2]An error occurred (SignatureDoesNotMatch) when calling the GetSessionToken operation: The request signature we calculated does not match the signature you provided. Check your AWS Secret Access Key and signing method. Consult the service documentation for details.

檢查~/.aws/config以及 ~/.credentials 內 Access keys 是否跟輸入值相同

[3]git 'remote-codecommit' is not a git command。參見 'git --help'。
檢查是否有安裝 git-remote-codecommit
查看	$PATH的路徑，看裡面是否有 git-remote-codecommit
MAC出現這個問題，最後是透過sudo pip3 install git-remote-codecommit 安裝才能使用

[4]測試時反覆測試會有問題
要在鑰匙圈存取把有獲得的key刪除掉

[5]設定AWS_ACCESS_KEY_ID、AWS_SECRET_ACCESS_KEY、AWS_SESSION_TOKEN後仍然無法使用
注意
1.每次取得的AWS_ACCESS_KEY_ID會不一樣，記得要換
2.可以用echo $AWS_ACCESS_KEY_ID 來檢查是否將變數輸入
3.在AWS_ACCESS_KEY_ID＝後不能留空格，不然會輸入空值

[6]有設定以下東西，不過忘記是什麼了
```
git config --global credential.helper '!aws --profile bing-test codecommit credential-helper $@'
git config --global credentials.helper UseHttpPath=true
```

## 參考資料：
[1]Setup steps for HTTPS connections to AWS CodeCommit with git-remote-codecommit
https://docs.aws.amazon.com/codecommit/latest/userguide/setting-up-git-remote-codecommit.html

[2]How do I use an MFA token to authenticate access to my AWS resources through the AWS CLI?
https://repost.aws/knowledge-center/authenticate-mfa-cli

## 附件
[1]IAM policy
```
{
	"Version": "2012-10-17",
	"Statement": [
		{
			"Sid": "AllowManageOwnPasswords",
			"Effect": "Allow",
			"Action": [
				"iam:ChangePassword",
				"iam:GetUser",
				"iam:GetLoginProfile",
				"iam:UpdateLoginProfile"
			],
			"Resource": "arn:aws:iam::*:user/${aws:username}"
		},
		{
			"Sid": "BlockMostAccessUnlessSignedInWithMFA",
			"Effect": "Deny",
			"NotAction": [
				"iam:CreateVirtualMFADevice",
				"iam:EnableMFADevice",
				"iam:ListMFADevices",
				"iam:ListUsers",
				"iam:ListVirtualMFADevices",
				"iam:ResyncMFADevice",
				"iam:GetAccountPasswordPolicy"
			],
			"Resource": "*",
			"Condition": {
				"BoolIfExists": {
					"aws:MultiFactorAuthPresent": "false"
				}
			}
		}
	]
}

```

















