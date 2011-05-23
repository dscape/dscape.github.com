---
layout: post
title: RSA Encrypt & Decrypt in ruby
---
# RSA Encrypt & Decrypt in ruby

Well I finished the encrypt with RSA on ruby some hours ago and felt like 
[sharing](http://github.com/dscape/rudolph/tree/e94b7f106afe862cd99e842dc11d698a5117a64a/src/rudolph/crypt.rb). :)

![Rudolph](http://img.skitch.com/20081208-c9fba7g19he82fmfg28g6wma28.png)

Case you feel like doing something back for me just [download the latest release](http://github.com/dscape/rudolph/zipball/e94b7f106afe862cd99e842dc11d698a5117a64a) of my beta twitter client and send me some comments to my email. It's pretty hard to test something when my environment is completely contaminated!

     require 'openssl'
     require 'Base64'
     
     class Rudolph
       class Crypt
         def initialize data_path
           @data_path = data_path
           @private   = get_key 'id_rsa'
           @public    = get_key 'id_rsa.pub'
         end
     
         def encrypt_string message
           Base64::encode64(@public.public_encrypt(message)).rstrip
         end
     
         def decrypt_string message
           @private.private_decrypt Base64::decode64(message)
         end
     
         def self.generate_keys data_path
           rsa_path = File.join(data_path, 'rsa')
           privkey  = File.join(rsa_path, 'id_rsa')
           pubkey   = File.join(rsa_path, 'id_rsa.pub')
           unless File.exists?(privkey) || File.exists?(pubkey)
             keypair  = OpenSSL::PKey::RSA.generate(1024)
             Dir.mkdir(rsa_path) unless File.exist?(rsa_path)
             File.open(privkey, 'w') { |f| f.write keypair.to_pem } unless File.exists? privkey
             File.open(pubkey, 'w') { |f| f.write keypair.public_key.to_pem } unless File.exists? pubkey
           end
         end
     
         private
         def get_key filename
           OpenSSL::PKey::RSA.new File.read(File.join(@data_path, 'rsa', filename))
         end
       end
     end
