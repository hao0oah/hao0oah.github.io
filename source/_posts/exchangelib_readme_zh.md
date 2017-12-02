### 用法
一些exchangelib如何使用的例子：

####设置和连接

```
from datetime import timedelta
from exchangelib import DELEGATE, IMPERSONATION, Account, Credentials, ServiceAccount, \
    EWSDateTime, EWSTimeZone, Configuration, NTLM, CalendarItem, Message, \
    Mailbox, Attendee, Q, ExtendedProperty, FileAttachment, ItemAttachment, \
    HTMLBody, Build, Version

# 定义你的凭证，用户名格式通常是WINDOMAIN\username，WINDOMAIN是你要连接到的Windows域用户名，有些服务器也会接受PrimarySMTPAddress('myusername@example.com')格式(Office365需要它)，如果你的服务器支持那么也可以是UPN格式。
credentials = Credentials(username='MYWINDOMAIN\\myusername', password='topsecret')

# 如果你打算长时间运行，你可能需要启用容错(fault-tolerance)功能。容错就是在服务器不可用或者返回错误信息时，请求服务器的次数会进行指数退避，并在达到一定的阈值时放弃请求。这可以防止自动化脚本压垮一个存在缺陷或者超载的服务器，或者经常发生在大型Exchange设备中的隐性的间歇服务中断。

# 如果你想要启用容错(fault-tolerance)功能，那么以service account方式来创建凭证。
credentials = ServiceAccount(username='FOO\\bar', password='topsecret')

# Account是你想要连接到的Exchange服务器上的账户，可以是与你连接的凭证相关联的账户，或已经授权访问权限的服务器上的其他任何账户。
# 'primary_smtp_address'是主SMTP地址分配的账户，如果你使用了自动发现，那么别名地址也可以使用，这样的话'Account.primary_smtp_address'将设置为主SMTP地址。
my_account = Account(primary_smtp_address='myusername@example.com', credentials=credentials,
                     autodiscover=True, access_type=DELEGATE)
johns_account = Account(primary_smtp_address='john@example.com', credentials=credentials,
                        autodiscover=True, access_type=DELEGATE)
marys_account = Account(primary_smtp_address='mary@example.com', credentials=credentials,
                        autodiscover=True, access_type=DELEGATE)
still_marys_account = Account(primary_smtp_address='alias_for_mary@example.com',
                              credentials=credentials, autodiscover=True, access_type=DELEGATE)

# 设置一个目标账户，开启一个自动发现目标EWS端点的查找任务
account = Account(primary_smtp_address='john@example.com', credentials=credentials,
                  autodiscover=True, access_type=DELEGATE)

# 如果你的凭证已经被模拟访问目标账户，请设置不同的'access_type':
account = Account(primary_smtp_address='john@example.com', credentials=credentials,
                  autodiscover=True, access_type=IMPERSONATION)

# 如果服务器不支持自动发现，或者你想避免自动发现的开销
# 那么可以使用Configuration对象来设置服务器地址
config = Configuration(server='mail.example.com', credentials=credentials)
account = Account(primary_smtp_address='john@example.com', config=config,
                  autodiscover=False, access_type=DELEGATE)

# 'exchangelib'将尝试猜测服务器的版本和认证方法。如果你使用了奇怪或锁定的安装版本导致猜测失败，你可以设置明确的认证方法和版本：
version = Version(build=Build(15, 0, 12, 34))
config = Configuration(server='example.com', credentials=credentials, version=version, auth_type=NTLM)

# 如果你连接经常相同的帐户，你可以缓存自动发现的结果，这样你可以跳过频繁的自动发现查找：
ews_url = account.protocol.service_endpoint
ews_auth_type = account.protocol.auth_type
primary_smtp_address = account.primary_smtp_address

# 5分钟后，取出缓存值并创建帐户而不是通过自动发现：
config = Configuration(service_endpoint=ews_url, credentials=credentials, auth_type=ews_auth_type)
account = Account(
    primary_smtp_address=primary_smtp_address, config=config, autodiscover=False, access_type=DELEGATE
)

# 如果你需要支持代理或自定义TLS验证，你可以提供自定义的'requests'传输适配器，查看描述文档： http://docs.python-requests.org/en/master/user/advanced/#transport-adapters
# 'exchangelib'提供了一个忽略SSL验证错误的适配器示例，如使用请自己承担风险。
from exchangelib.protocol import BaseProtocol, NoVerifyHTTPAdapter
BaseProtocol.HTTP_ADAPTER_CLS = NoVerifyHTTPAdapter
```

#### 文件夹

```
# 最常见的可用文件夹如：account.calendar, account.trash, account.drafts, account.inbox,account.outbox, account.sent, account.junk, account.tasks, and account.contacts.
# 有多种方式来导航文件夹树和搜索文件夹。如果你的文件夹名包含斜线，那么通配符和绝对路径访问时可能出现意想不到的结果。
some_folder.parent
some_folder.parent.parent.parent
some_folder.root  # 返回文件夹结构的根，任意级别。同Account.root
some_folder.children  # 子文件夹生成器
some_folder.absolute  # 返回绝对路径字符串
some_folder.walk()  # 可以返回任意深度级别的子文件夹生成器
# 通配符使用通用的UNIX通配符语法
some_folder.glob('foo*')  # 返回与模式匹配的子文件夹
some_folder.glob('*/foo')  # 返回子文件夹名为foo的所有子文件夹
some_folder.glob('**/foo')  # 返回所有任意深度的名为foo的子文件夹
some_folder / 'sub_folder' / 'even_deeper' / 'leaf'  # 跟pathlib.Path一样
some_folder.parts  # 返回some_folder，some_folder.parent，some_folder.parent.parent文件夹实例
# tree() 返回给定深度的文件树结构的字符串形式。
print(root.tree())
'''
root
├── inbox
│   └── todos
└── archive
    ├── Last Job
    ├── exchangelib issues
    └── Mom
'''

# Folders有一些有用的计数器:
account.inbox.total_count
account.inbox.child_folder_count
account.inbox.unread_count
# 更新计数器
account.inbox.refresh()
# 自动缓存第一次访问过的文件夹树结构，如要清除缓存，刷新根文件夹：
account.root.refresh()
```

#### 日期，时间和时区
EWS对日期和时区有一些特殊要求。当你用到日期时间时，你需要使用特定的EWSDate，EWSDatetime和EWSTimeZone类。

```
# EWSTimeZone的使用就像pytz.timezone()一样
tz = EWSTimeZone.timezone('Europe/Copenhagen')
# 你也可以使用你操作系统中定义的本地时区
tz = EWSTimeZone.localzone()

# EWSDate和EWSDateTime的使用就像datetime.datetime和datetime.date一样. 通过EWSTimeZone.localize()来创建timezone-aware和datetimes:
localized_dt = tz.localize(EWSDateTime(2017, 9, 5, 8, 30))
right_now = tz.localize(EWSDateTime.now())

# Datetime的数学计算是透明的
two_hours_later = localized_dt + timedelta(hours=2)
two_hours = two_hours_later - localized_dt

# Dates
my_date = EWSDate(2017, 9, 5)
today = EWSDate.today()
also_today = right_now.date()

# UTC helpers. 'UTC' is the UTC timezone as an EWSTimeZone instance.
# 'UTC_NOW' returns a timezone-aware UTC timestamp of current time.
from exchangelib import UTC, UTC_NOW

right_now_in_utc = UTC.localize(EWSDateTime.now())
right_now_in_utc = UTC_NOW()
```

#### 创建、更新、删除、发送和移动

```
# 有一个在用户的标准日历创建日历条目的例子，如果你想访问非标准日历，从account.folders[Calendar]选择不同的日历。
#
# 您可以创建、更新和删除单个项目:
from exchangelib.items import SEND_ONLY_TO_ALL, SEND_ONLY_TO_CHANGED
item = CalendarItem(folder=account.calendar, subject='foo')
item.save()  # 这将为item创建'item_id'和'changekey'值
item.save(send_meeting_invitations=SEND_ONLY_TO_ALL)  # 发送会议邀请给与会者
# 更新字段。所有字段都必须使用相应的Python类型。
item.subject = 'bar'
# 打印'CalendarItem'类所有可用字段，注意某些只读的字段，或者在保存或发送项目后的只读字段，和某些在旧版本Exchange中不支持字段。
print(CalendarItem.FIELDS)
item.save()  # 当item有'item_id'时将更新该item
item.save(update_fields=['subject'])  # 仅更新指定字段
item.save(send_meeting_invitations=SEND_ONLY_TO_CHANGED)  # 发送会议邀请,只发送给有变化的与会者
item.delete()  # 彻底删除
item.delete(send_meeting_cancellations=SEND_ONLY_TO_ALL)  # 会议取消发送给所有与会人员
item.soft_delete()  # 删除,但在恢复文件夹中保留副本
item.move_to_trash()  # 移动到垃圾文件夹
item.move(account.trash)  # 也是移动到垃圾文件夹

# 如果不想让本地产生副本，你可以这样发送电子邮件:
m = Message(
    account=a,
    subject='Daily motivation',
    body='All bodies are beautiful',
    to_recipients=[Mailbox(email_address='anne@example.com'), Mailbox(email_address='bob@example.com')],
    cc_recipients=['carl@example.com', 'denice@example.com'],  # 简单的字符串也可以
    bcc_recipients=[Mailbox(email_address='erik@example.com'), 'felicity@example.com'],  # 或者两者的混合
)
m.send()

# 否则，如果你想在比如'Sent'文件夹保留副本：
m = Message(
    account=a,
    folder=a.sent,
    subject='Daily motivation',
    body='All bodies are beautiful',
    to_recipients=[Mailbox(email_address='anne@example.com')]
)
m.send_and_save()

# EWS区别于纯文本和HTML内容.如果要发送HTML正文内容，请使用HTMLBody帮手。客户端将此视为HTML并正确显示内容
item.body = HTMLBody('<html><body>Hello happy <blink>OWA user!</blink></body></html>')
```


#### 搜索