aws s3 rm s3://hyundai-translator/logs2020-06-14-20-19-32-07257108170D96DE

aws s3 rm s3://hyundai-translator/ --include "logs2020-06-14"

aws s3 rm s3://hyundai-translator/ --exclude "*.apk"

aws s3 ls s3://hyundai-translator | grep "logs2020-06-08"

aws s3 ls "s3://hyundai-translator/logs2020-06-08"

aws s3 rm $(aws s3 ls "s3://hyundai-translator/logs2020-06-08" | awk '{print "s3://hyundai-translator/"$4}')

aws s3 rm s3://hyundai-translator --exclude "*"  --include "logs2021-04-08*" --dryrun

aws s3 rm s3://hyundai-translator/logs2020-06-08-11-23-40-9E5A0B4346AA71A --dryrun