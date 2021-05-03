## Vice-VeRSA 
### Points : *200*
### Description :
> L'administrateur se cache bien d'indiquer aux utilisateurs que leurs messages sont enregistrés. Prouvez-lui qu'on ne peut pas la faire à l'envers aux utilisateurs.


### Solution :
The name of the chall can make us think about a crypto chall. After testing the app, we quickly find that we get a JWT token when using the reverse string function of the site. The JWT token is in a cookie called `session=<JWT_TOKEN>`. This is an example cookie `eyJ0eXAiOiJKV1QiLCJhbGciOiJSUzI1NiIsImtpZCI6IjAifQ.eyJzdHJpbmciOiIiLCJyb2xlIjoiZ3Vlc3QifQ` (I didn't put the part about the signature), that can be decrypt as following  :
```json
{
  "typ": "JWT",
  "alg": "RS256",
  "kid": "0"
}
{
  "string": "test",
  "role": "guest"
}
```

The string parameter correspond to the chain that we want to reverse on the site : `test` become `tset`. We played a bit with `kid` and we see that this parameter can be 0, 1 or 2. In a comment of the source code there is a `/historique`, however we didn't have permission to access this part. We understand that it will be necessary to change our role to admin.

After getting all this information we try to use [jwt_tool](https://github.com/ticarpi/jwt_tool), in order to use the fuzzing part of the tool. We didn't get much success except that the kid can be set to empty and our token will still be valid but it doesn't allow us to access `/historique` part. Without the public key it will be difficult to do more attacks.
So we try to recover the RSA public key :
```python
# https://github.com/FlorianPicca/JWT-Key-Recovery
python3 recover.py eyJ0eXAiOiJKV1QiLCJhbGciOiJSUzI1NiIsImtpZCI6IjAifQ.eyJzdHJpbmciOiJhIiwicm9sZSI6Imd1ZXN0In0.<signature> eyJ0eXAiOiJKV1QiLCJhbGciOiJSUzI1NiIsImtpZCI6IjAifQ.eyJzdHJpbmciOiIiLCJyb2xlIjoiZ3Vlc3QifQ.<signature>
Recovering public key for algorithm RS256...
Found public RSA key !
n=26451045880964917513500384554170817361416002571109323756587984316856882166600328406421489096897898642651442847900199601304963538350047883183579130756781323269629194136721900337192506462030774874728607767647872177503355240337120882142783093099150509303327996012421752957488757794618331761757840545947319249000775787430347930747865590144543178555226128667553193702710789911878998342585553168381830185135508390340657676458670407304327894647689701103713000730792845731920670463877317625170112005192759353936549709813113349510590820755042322951273495963408223355841314599444712000223270748965159897835541243233949292926371
e=65537
-----BEGIN PUBLIC KEY-----
MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEA0YhbZGPINuB11x9dKdKu
xHqOV5Tpxu1/DKV1S9F/i7tmnw+92nDGa4Wi/MUnQdTCebc8JcVFLDMiUhuKPG0V
vAqXymh+fuLK4Puemy0EcTKhXLqjE5qaQNZcCHHVzZ4q6kpFttw34SfgvB3ijNwZ
46YSAC7HG8rEAJRRqtvS38MAZ0/lbrJY5FQHboLjMM/Xctwo/rn9zRdGLf+eTt/G
Du8P4zq4idqo017Cqcxtyqm28/NcGDDNuWRXXOg+qd/NEFzUckB8PyJKjLw9koky
7zMSjBdBfIydGSht4XHrg5HBh+7nwKLJAsDPvfD8Wqn1tfQgWiD9eD80U64bVB+V
owIDAQAB
-----END PUBLIC KEY-----

python3 recover.py eyJ0eXAiOiJKV1QiLCJhbGciOiJSUzI1NiIsImtpZCI6IjEifQ.eyJzdHJpbmciOiIiLCJyb2xlIjoiZ3Vlc3QifQ.s4CDRnqF7uTBf7xzgNtfVEP-SiA8av-1u7dNNi4bx7jtAKpE-PFsOMvVI8i-7UzAIigeN2JQD_LiAt7gKvUl7j4FVncpxHIwtAAyFuaHzOIVor-gyGLsIHsNKAKG6-PpdmFxIQgiTqA6RSNaG9ha02VOP5hY-7TpDNGjyZZnhWsU-jGrkvnQ4endyz41ugdxJdQWOFThNlfq7UrO3ZeFaZcGOs7yn0BIuipT8JLGFBm1ZEgKC8Lyar_mkXrNyV1B26YA8LsAqOuvFVdDwPUZJ_ykssjeXC_EN-JKIyaE5SrTeFnw1KSz96gpBuQBNAjnMbBvrzRHyG7VjYLnkVu4pA eyJ0eXAiOiJKV1QiLCJhbGciOiJSUzI1NiIsImtpZCI6IjEifQ.eyJzdHJpbmciOiJhIiwicm9sZSI6Imd1ZXN0In0.lO-qZtMT00qQ5IOUw3OcttQXt2QkntxC6aB_tpaEMZlVbkoxHXZfmCYFJVRZ3wHVLiWLY2MPczOLd9LHvNubrjNvfeQ8eJ5dd7YpP2c27iv9Lju0JrSvOAugqcpeze_ZheqlkaJdEwEi-S4KgNElnmv6VAqWDv3c-pqZku7DBZn1B2Pvu6B9vUFpD81pToIGWnh8fo7EEN6IDAmT4I7DMx1K2pOsFkMxSzSagPcSE6ktNKUjdZuGcJ7fm_ZBrzkPAK_w80Vpjhg4zAsHEeZMWpA4D8LbqV9LpdpFI54MBHAedAY-Zm3EGriVaJhEu_-jkTT-YX-ebiFZ4oOwjzDY9A
Recovering public key for algorithm RS256...
Found public RSA key !
n=23665652820134805374198160670285833921391080168347242686789486999501147452564182203815403432586037211580956807418390515260607797114130600470792877974783209468301203025857100674184179834125773001492324249995631518632190591284347199695615566699061802073984938509524118604141487334019736930099338674532167272834445210127085946363143475338489425347697379610470637836163978122048233036129066473971261891100314946785632986774865922264847742332593870916133974536065637794870077767216697298265895973548680864568583117925623450171604556505682160596979527747518700282565884296112666999750704513275226624446992913735981325611673
e=65537
-----BEGIN PUBLIC KEY-----
MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAu3fULUk+yOOQtvyYYJGU
XN+ZiHbJXJUO+1GSvDiw7cM9Ls6v+W+yHRGyWyTjKZJLIe9LgXDQ2gC5DNs5ZIHU
Om77Vui7xgI4vPhtFRxz3MFaIamI9FM4q9ucoSuhXMElZa4KVt/KBEapMvLs2TTl
1ri8CzL4yA2zWo0ASVTUfasSfVKyzQRZvBP5aFmgImQf3CVCzwVyttgb09Wjugxc
9wjM2nU3aq5Egz5vbIxzsSBjnGqy3Zhh3CUnC0Xxoj7Qk/p72hOzGIxgsYp6blY0
Dx12F2zyVa4zUb6heNG1SmYTSBqeaijt0KP9sk1yxMw5jTtxWXjjW3ez7IKVJ6fy
mQIDAQAB
-----END PUBLIC KEY-----

python3 recover.py eyJ0eXAiOiJKV1QiLCJhbGciOiJSUzI1NiIsImtpZCI6IjIifQ.eyJzdHJpbmciOiJhIiwicm9sZSI6Imd1ZXN0In0.<signature>
eyJ0eXAiOiJKV1QiLCJhbGciOiJSUzI1NiIsImtpZCI6IjIifQ.eyJzdHJpbmciOiIiLCJyb2xlIjoiZ3Vlc3QifQ.<signature>
Recovering public key for algorithm RS256...
Found public RSA key !
n=22731365669722716790523988469211174698469647719778831863970396221572599409300719240441972167971859096690301602182969197363505496123904858731217310323069800081347653770883702578911025506738080849436708669671186348533493041292769716064576663922411644860482974431829677048235872088045298198687332564790708320625966491365742866664943328632514191628724244741007713325357654856556937713627169871880861184843650059723398603225879352578010649653414401635886489704537350073827506117761008999068169633867812485790098429368696301604341046865861665894241805556589392944953338821552542037920683039404461145689037243695687228406013
e=65537
-----BEGIN PUBLIC KEY-----
MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAtBEtj9sd9uHDddLcG5cb
V9fXJCbBDn07Umgb57q6zrJvjjYp03+PFOyZjVG4D0Aitmi+8FiP26V7CpdzWCUB
Iye8ARqwVFGywuUYzWYIrHRdy1okqpz456uud3zXCS8bFkRAsCetlmpWezBTZ79X
rUcsC3Y/RHHMlItAweTj/NihFM7PluEgFgmz+ScHAt4vGQb1HRAwfqONmIQ0QplK
aofQPewmQWzk6jZ37PNPEER50VUyRDgLUj7sL8uFOD6sF8raFxsjYzKyat083ATQ
GD2BcrbHh6jYBJDeZixrKQnF66rvPaWYL815fkp+jwANnxLz79JW0FzRqyWcDZ84
/QIDAQAB
-----END PUBLIC KEY-----
```
We have tried several attacks with [RsaCtfTool](https://github.com/Ganapati/RsaCtfTool) without success. After this, I got stuck for a while. Then I thought that the public keys could have been generated with an identical p or q. Liveoverflow made a video about this type of attacks [here](https://www.youtube.com/watch?v=sYCzu04ftaY).
```python
# https://www.youtube.com/watch?v=sYCzu04ftaY
from Crypto.PublicKey import RSA
from Crypto.Util.number import GCD, inverse
import itertools

n = [
RSA.importKey(open('key0.pub', 'r').read()).n,
RSA.importKey(open('key1.pub', 'r').read()).n,
RSA.importKey(open('key2.pub', 'r').read()).n
]
e = 65537

for i in range(len(n)):
    g = GCD(n[i], n[(i % (len(n)-1))+1])
    if g > 1:
        p = g
        q = n[i] // p # / is floating point division, and // is floor/integer division
        phi = (p - 1) * (q - 1)
        d = inverse(e, phi)

        #print(f"e: {e}")
        #print(f"n: {n[i]}")
        #print(f"p: {p}")
        #print(f"q: {q}")
        #print(f"phi: {phi}")
        #print(f"d: {d}\n")
        print(f"Generating private key")
        with open(f'key{i}.priv', 'wb') as f:
            f.write(RSA.construct((n[i], e, d)).exportKey())
        print(f"key{i}.priv has been created\n")
```
By printing **p** we see that this one is the greatest common divisor of the 3 keys.

Nice now we have public + private key, we load a session cookie with kid 0 on [jwt.io](https://jwt.io), we use public and private key of this kid, however that not work (for some reason i don't know). Never give up ! I try with the two other kid (public key + private key + cookie) and it works for both !

Flag : `FCSC{e1f444434b8c52a812e6dd0f59b71c32253018473384476feacc2fc9eefdc7be}`

