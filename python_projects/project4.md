## At work, we have an internal application running on postgressql, and we need a python script that will take backup of the database and upload it in s3 bucket everyday at a certain time.

### The script is below:

Before executing the below script, you have to set Database credentials, AWS Access key credentials and backup path.

```
import os
import sys
import subprocess
from optparse import OptionParser
from datetime import datetime

import boto3
from boto3.s3 import Key


DB_USER = 'databaseuser'
DB_NAME = 'databasename'

BACKUP_PATH = r'/webapps/myapp/db_backups'

FILENAME_PREFIX = 'myapp.backup'

# Amazon S3 settings.
AWS_ACCESS_KEY_ID = os.environ['AWS_ACCESS_KEY_ID']
AWS_SECRET_ACCESS_KEY = os.environ['AWS_SECRET_ACCESS_KEY']
AWS_BUCKET_NAME = 'myapp-db-backups'


def main():
    parser = OptionParser()
    parser.add_option('-t', '--type', dest='backup_type',
                      help="Specify either 'hourly' or 'daily'.")

    now = datetime.now()

    filename = None
    (options, args) = parser.parse_args()
    if options.backup_type == 'hourly':
        hour = str(now.hour).zfill(2)
        filename = f'{FILENAME_PREFIX}.h{hour}'
    elif options.backup_type == 'daily':
        day_of_year = str(now.timetuple().tm_yday).zfill(3)
        filename = f'{FILENAME_PREFIX}.d{day_of_year}'
    else:
        parser.error('Invalid argument.')
        sys.exit(1)

    destination = r'%s/%s' % (BACKUP_PATH, filename)

    print (f'Backing up {DB_NAME} database to {destination}')
    ps = subprocess.Popen(
        ['pg_dump', '-U', DB_USER, '-Fc', DB_NAME, '-f', destination],
        stdout=subprocess.PIPE
    )
    output = ps.communicate()[0]
    for line in output.splitlines():
        
        print(line)

    print(f'Uploading {filename} to Amazon S3...')
    upload_to_s3(destination, filename)


def upload_to_s3(source_path, destination_filename):
    """
    Upload a file to an AWS S3 bucket.
    """
    conn = boto3.connect_s3(AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY)
    bucket = conn.get_bucket(AWS_BUCKET_NAME)
    k = Key(bucket)
    k.key = destination_filename
    k.set_contents_from_filename(source_path)


if __name__ == '__main__':
    main()
    
```
