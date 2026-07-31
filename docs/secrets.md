 cloudflare-api-token · cert-manager · Cloudflare-Token (Zone-DNS-Edit, damians.cloud) · CREATE IN cf
 kubectl create secret generic cloudflare-api-token -n cert-manager --from-literal=api-token=

 kubectl create secret generic paperless-secret `
>>   -n media `
>>   --from-literal=PAPERLESS_SECRET_KEY=""