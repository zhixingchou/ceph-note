```
# -*- coding: UTF-8 -*-
#  dd if=/dev/urandom of=64M.txt bs=64M count=1  #生成64M大小的随机文件
#  split --bytes 32M --numeric-suffixes --suffix-length=3 64M.txt 64M.txt.  #将64M.txt 文件切为2个32M大小的文件
#  pip install xmltodict
import base64
import json
import requests
import logging
from requests_toolbelt.utils import dump
logging.basicConfig(level=logging.DEBUG)

import requests
import xmltodict
from xmltodict import parse, unparse, OrderedDict


import hmac

from hashlib import sha1 as sha

py3k = False
try:
    from urlparse import urlparse, unquote
    from base64 import encodestring
except:
    py3k = True
    from urllib.parse import urlparse, unquote
    from base64 import encodebytes as encodestring

from email.utils import formatdate

from requests.auth import AuthBase


class S3Auth(AuthBase):
    """Attaches AWS Authentication to the given Request object."""

    service_base_url = 's3.amazonaws.com'
    # List of Query String Arguments of Interest
    special_params = [
        'acl', 'location', 'logging', 'partNumber', 'policy', 'requestPayment',
        'torrent', 'versioning', 'versionId', 'versions', 'website', 'uploads',
        'uploadId', 'response-content-type', 'response-content-language',
        'response-expires', 'response-cache-control', 'delete', 'lifecycle',
        'response-content-disposition', 'response-content-encoding', 'tagging',
        'notification', 'cors', 'syncing'
    ]

    def __init__(self, access_key, secret_key, service_url=None):
        if service_url:
            self.service_base_url = service_url
        self.access_key = str(access_key)
        self.secret_key = str(secret_key)

    def __call__(self, r):
        # Create date header if it is not created yet.
        if 'date' not in r.headers and 'x-amz-date' not in r.headers:
            r.headers['date'] = formatdate(
                timeval=None,
                localtime=False,
                usegmt=True)
        signature = self.get_signature(r)
        if py3k:
            signature = signature.decode('utf-8')
        r.headers['Authorization'] = 'AWS %s:%s' % (self.access_key, signature)
        return r

    def get_signature(self, r):
        canonical_string = self.get_canonical_string(
            r.url, r.headers, r.method)
        if py3k:
            key = self.secret_key.encode('utf-8')
            msg = canonical_string.encode('utf-8')
        else:
            key = self.secret_key
            msg = canonical_string
        h = hmac.new(key, msg, digestmod=sha)
        return encodestring(h.digest()).strip()

    def get_canonical_string(self, url, headers, method):
        parsedurl = urlparse(url)
        objectkey = parsedurl.path[1:]
        query_args = sorted(parsedurl.query.split('&'))

        bucket = parsedurl.netloc[:-len(self.service_base_url)]
        if len(bucket) > 1:
            # remove last dot
            bucket = bucket[:-1]

        interesting_headers = {
            'content-md5': '',
            'content-type': '',
            'date': ''}
        for key in headers:
            lk = key.lower()
            try:
                lk = lk.decode('utf-8')
            except:
                pass
            if headers[key] and (lk in interesting_headers.keys()
                                 or lk.startswith('x-amz-')):
                interesting_headers[lk] = headers[key].strip()

        # If x-amz-date is used it supersedes the date header.
        if not py3k:
            if 'x-amz-date' in interesting_headers:
                interesting_headers['date'] = ''
        else:
            if 'x-amz-date' in interesting_headers:
                interesting_headers['date'] = ''

        buf = '%s\n' % method
        for key in sorted(interesting_headers.keys()):
            val = interesting_headers[key]
            if key.startswith('x-amz-'):
                buf += '%s:%s\n' % (key, val)
            else:
                buf += '%s\n' % val

        # append the bucket if it exists
        if bucket != '':
            buf += '/%s' % bucket

        # add the objectkey. even if it doesn't exist, add the slash
        buf += '/%s' % objectkey

        params_found = False

        # handle special query string arguments
        for q in query_args:
            k = q.split('=')[0]
            if k in self.special_params:
                buf += '&' if params_found else '?'
                params_found = True

                try:
                    k, v = q.split('=', 1)

                except ValueError:
                    buf += q
                else:
                    buf += '{key}={value}'.format(key=k, value=unquote(v))

        return buf


host = '127.0.0.1:8000'
access_key = 'yly'
secret_key = 'yly'
bucketname = 'test1'
objectname = 'multiobj'

#申请uploadId
cmd = '/%s/%s?uploads' % (bucketname,objectname)
url = 'http://%s%s' % (host,cmd)
response = requests.post(url, auth=S3Auth(access_key, secret_key,service_url=host))
UploadId = xmltodict.parse(response.content)['InitiateMultipartUploadResult']['UploadId']

cmd = '/%s/%s?partNumber=1&uploadId=%s' % (bucketname,objectname,UploadId)
host = '127.0.0.1:8000'
url = 'http://%s%s' % (host,cmd)
with open('10M.txt.000', 'rb') as fin:
    data = fin.read()
response = requests.put(url, auth=S3Auth(access_key, secret_key,service_url=host),data=data)

cmd = '/%s/%s?partNumber=2&uploadId=%s' % (bucketname,objectname,UploadId)
host = '127.0.0.1:8000'
url = 'http://%s%s' % (host,cmd)
with open('10M.txt.001', 'rb') as fin:
    data = fin.read()
response = requests.put(url, auth=S3Auth(access_key, secret_key,service_url=host),data=data)

cmd = '/%s/%s?uploadId=%s' % (bucketname,objectname,UploadId)
url = 'http://%s%s' % (host,cmd)
response = requests.get(url, auth=S3Auth(access_key, secret_key,service_url=host))
obj2 = []
for i in xmltodict.parse(response.content)['ListPartsResult']['Part']:
    obj2.append({'PartNumber': i['PartNumber'], 'ETag': i['ETag']})
obj = {'CompleteMultipartUpload': OrderedDict((
    ('Part',obj2),
))}

completeme = xmltodict.unparse(obj,full_document=False)

cmd = '/%s/%s?uploadId=%s' % (bucketname,objectname,UploadId)
url = 'http://%s%s' % (host,cmd)

callback_dict = {}
callback_dict['callbackUrl'] = 'http://67.218.159.42:23450'
# callback_dict['callbackHost'] = 'oss-cn-hangzhou.aliyuncs.com'
callback_dict['callbackBody'] = 'bucket=${bucket}&filename=${object}&size=${size}&mimeType=${mimeType}&my_var=${x:var1}'
callback_dict['callbackBodyType'] = 'application/x-www-form-urlencoded'
callback_param = json.dumps(callback_dict).strip()
base64_callback_body = base64.b64encode(callback_param)
headers={'x-amz-meta-callback': base64_callback_body}
callback_var = {}
callback_var["x:var1"]='value1'
callback_var["x:var2"]='value2'
callback_var_param = json.dumps(callback_var).strip()
base64_callback_var = base64.b64encode(callback_var_param)
headers['x-amz-meta-callback-var'] = base64_callback_var

response = requests.post(url, auth=S3Auth(access_key, secret_key,service_url='127.0.0.1:8000'),headers=headers, data=completeme)
data = dump.dump_all(response)
print(data.decode('utf-8'))

```

use boto3
```
pip uninstall botocore
#安装支持回调的botocore
pip install https://github.com/joke-lee/botocore/archive/1.0.zip
```


```
import base64
import json

callback_dict = {}
callback_dict['callbackUrl'] = 'http://10.254.9.35:23450/index.php?id=1&index=2'
callback_dict['callbackBody'] = 'bucket=${bucket}&filename=${object}&size=${size}&mimeType=${mimeType}&my_var=${x:var1}'
callback_dict['callbackBodyType'] = 'application/x-www-form-urlencoded'

callback_var = {}
callback_var["x:var1"]='value1'
callback_var["x:var2"]='value2'


callback_param = json.dumps(callback_dict).strip()
base64_callback_body = base64.b64encode(callback_param)
headers={'x-amz-meta-callback': base64_callback_body}

callback_var_param = json.dumps(callback_var).strip()
base64_callback_var = base64.b64encode(callback_var_param)
headers['x-amz-meta-callback-var'] = base64_callback_var

from boto3.session import Session
import boto3
access_key = "eos"
secret_key = "eos"
url = "http://10.254.3.68"
session = Session(access_key, secret_key)
s3_client = session.client('s3', endpoint_url=url,config = boto3.session.Config( signature_version = 's3v4') )
mpu = s3_client.create_multipart_upload(Bucket="callback",Key="yyy")
part_info = {
    'Parts': []
}
i = 1
file = open("5M","rb")
while 1:
    data = file.read(5 * 1024 * 1024)
    if data == b'':
        break
    response = s3_client.upload_part(Bucket="callback", Key="yyy", PartNumber=i, UploadId=mpu["UploadId"],Body=data)
    part_info['Parts'].append({
        'PartNumber': i,
        'ETag': response['ETag']
    })
    i += 1

response = s3_client.complete_multipart_upload(Bucket="callback", Key="yyy", UploadId=mpu["UploadId"], MultipartUpload=part_info, Metadata={'callback': base64_callback_body,'callback-var': base64_callback_var} )
print response
```

```
import base64
import json

callback_dict = {}
callback_dict['callbackUrl'] = 'http://10.254.9.35:23450/index.php?id=1&index=2'
callback_dict['callbackBody'] = 'bucket=${bucket}&filename=${object}&size=${size}&mimeType=${mimeType}&my_var=${x:var1}'
callback_dict['callbackBodyType'] = 'application/x-www-form-urlencoded'

callback_var = {}
callback_var["x:var1"]='value1'
callback_var["x:var2"]='value2'


callback_param = json.dumps(callback_dict).strip()
base64_callback_body = base64.b64encode(callback_param)
headers={'x-amz-meta-callback': base64_callback_body}

callback_var_param = json.dumps(callback_var).strip()
base64_callback_var = base64.b64encode(callback_var_param)
headers['x-amz-meta-callback-var'] = base64_callback_var

from boto3.session import Session
import boto3
access_key = "eos"
secret_key = "eos"
url = "http://10.254.3.68"
session = Session(access_key, secret_key)
s3_client = session.client('s3', endpoint_url=url,config = boto3.session.Config( signature_version = 's3v4') )
response = s3_client.put_object(Bucket="callback",Key="xxxxxxxxxxxx", Metadata={'callback': base64_callback_body, 'callback-var': base64_callback_var }, Body="1234")
print response

```
