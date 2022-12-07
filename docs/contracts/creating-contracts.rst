.. index:: ! contract;creation, constructor

**************************
Akıllı Sözleşme Oluşturma
**************************

Akıllı sözleşmeler iki şekilde oluşturulabilir; "dışarıdan" bir Ethereum transactionı ile veya
Solidity kullanarak direkt başka bir akıllı sözleşme içerisinde.

`Remix <https://remix.ethereum.org/>`_ gibi IDE'ler oluşturma aşamasını kullanıcı arayüzü kullanarak kolayca gerçekleştirmenize yardımcı olur.

Programlama ile akıllı sözleşme oluşturmanın bir yolu ise `web3.js <https://github.com/ethereum/web3.js>`_ gibi bir JavaScript API kullanımıdır.
Akıllı sözleşme oluşturmaya yarayan `web3.eth.Contract <https://web3js.readthedocs.io/en/1.0/web3-eth-contract.html#new-contract>`_ isimli bir methodu vardır.

Bir akıllı sözleşme oluşturulduğu zaman :ref:`constructor <constructor>` isimli bir fonksiyon sadece bir kere olmak üzere çalıştırılır.
Bundan sonra bu fonksiyona erişim mümkün değildir.

Constructor kullanmak zorunlu değildir. Bir akıllı sözleşmede sadece bir adet constructor olabilir ve overloading
yapılması mümkün değildir.

Constructor çalıştırıldıktan sonra akıllı sözleşme kodunun son hali blok zincirinde saklanır. Bu kod
bütün public ve external fonksiyonları içerir. Deploy edilen kod constructor fonksiyonu ve
sadece constructor içerisinde çağrılan internal fonksiyonları içermez.

.. index:: constructor;arguments

Özünde constructor parametreleri :ref:`ABI encoded <ABI>` olarak akıllı sözleşme kodunun sonuna eklenir
(bytecode halinin), ama eğer ``web3.js`` kullanıyorsanız bunu umursamanıza gerek yok. Çünkü o
sizin için bu işlemleri gerçekleştiriyor.

Eğer bir akıllı sözleşme başka bir akıllı sözleşme oluşturmak istiyorsa, oluşturmak istediği akıllı sözleşmenin
kaynak kodunu (ve binary halini) bilmelidir. Bu demektir ki döngüsel olarak akıllı sözleşme oluşturmak
mümkün değildir.

.. code-block:: solidity

    // SPDX-License-Identifier: GPL-3.0
    pragma solidity >=0.4.22 <0.9.0;


    contract OwnedToken {
        // `TokenCreator` aşağıda belirtilmiş bir akıllı sözleşme tipidir.
        // Yeni bir akıllı sözleşme oluşturmak için kullanılmadığı sürece
        // referans etmekte sorun yoktur.
        TokenCreator creator;
        address owner;
        bytes32 name;

        // Burası constructor fonksiyonumuz. Burada
        // belirtilen isim ve akıllı sözleşmeyi oluşturan adres
        // akıllı sözleşmede kaydedilir.
        constructor(bytes32 name_) {
            // State değişkenlerine isimleri kullanılarak
            // erişilir. `this.owner` şeklinde bir kullanım
            // ile değil. Fonksiyonlara direkt olarak kendi
            // isimlerini kullanarak veya `this.f` şeklinde
            // bir kullanım ile erişebiliriz. Ancak ikinci
            // şekildeki kullanım external olarak (dışarıdan)
            // bir görüş sağlar. Özellikle constructorlarda,
            // fonksiyonlara external olarak erişmemelisiniz.
            // Çünkü o fonksiyonlar henüz oluşturulmadı, yani
            // erişilebilir değil.
            // Daha fazlası için bir sonraki bölüme bakabilirsiniz.
            owner = msg.sender;

            // Burada `address` tipinden `TokenCreator` tipine
            // bir explicit (açık) dönüşüm sağlarız ve bu fonksiyonu
            // çağıran akıllı sözleşmenin bir `TokenCreator` olduğunu varsayarız.
            // Bunu doğrulamanın gerçek bir yöntemi bulunmamakta.
            // Bu işlem yeni bir akıllı sözleşme oluşturmaz.
            creator = TokenCreator(msg.sender);
            name = name_;
        }

        function changeName(bytes32 newName) public {
            // Sadece `creator` `name` değişkenini değiştirebilir.
            // Akıllı sözleşmenin adresini explicit bir dönüşüm ile
            // elde edebilir ve karşılaştırmamızı yapabiliriz.
            if (msg.sender == address(creator))
                name = newName;
        }

        function transfer(address newOwner) public {
            // Sadece şu anki `owner` token transferi gerçekleştirebilir.
            if (msg.sender != owner) return;

            // `creator` adresindeki akıllı sözleşmenin bir fonksiyonunu
            // kullanarak, işlemin gerçekleştirilebilirliğini
            // kontrol edebilir. Eğer bu işlem hata verirse
            // (örneğin, out-of-gas (gazın tükenmesi)),
            // işlem burada son bulur.
            if (creator.isTokenTransferOK(owner, newOwner))
                owner = newOwner;
        }
    }


    contract TokenCreator {
        function createToken(bytes32 name)
            public
            returns (OwnedToken tokenAddress)
        {
            // Yeni bir `Token` akıllı sözleşmeyi oluşturur ve adresini return eder.
            // JavaScript tarafında return tipi `address` tipidir.
            return new OwnedToken(name);
        }

        function changeName(OwnedToken tokenAddress, bytes32 name) public {
            // `tokenAddress` isimli parametrenin tipi
            // `address` tipindendir.
            tokenAddress.changeName(name);
        }

        // Bir transferin gerçekleşip gerçekleşmeyeceğini belirler
        function isTokenTransferOK(address currentOwner, address newOwner)
            public
            pure
            returns (bool ok)
        {
            // Keyfi bir koşul ile işlemin gerçekleşip gerçekleşmeyeceğini
            // belirler ve sonucu return eder.
            return keccak256(abi.encodePacked(currentOwner, newOwner))[0] == 0x7f;
        }
    }
