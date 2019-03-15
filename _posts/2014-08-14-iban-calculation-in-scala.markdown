---
layout: post
title:  "IBAN Calculation in scala"
date:   2014-08-14
categories: [development]
tags: [scala, iban, tip]
---
Curiosity led me to the question: "how is the control number for an IBAN number calculated?"

I found the answer on (how could it not) [wikipedia](http://nl.wikipedia.org/wiki/International_Bank_Account_Number)

10 minutes later i had written a small scala function that calculates it

``` scala 

def createIBAN(bank:String, country:String, accountNr:String) = {

  //translate letters to numbers A=10, B=11 etc.
  def translateLettersToNumbers(text: String):String = {
    text.map(c => {
      if (c.toInt < 64) c.toString
      else (c.toInt - 55).toString
    }).mkString
  }

  // make sure account is 10 digits
  val accountPaddedTo10 = accountNr.reverse.padTo(10, "0").reverse.mkString
  // add bank before and country followed by 00 after
  val numeric = translateLettersToNumbers(bank + accountPaddedTo10 + country + "00")
  //controlnumber is the remainder of dividing the number by 97 subtracted from 98
  val controlNumber = (98-BigDecimal(numeric) % 97).toString

  //re-arrange and translate text to numbers again for validation
  val check = translateLettersToNumbers(bank + accountPaddedTo10 + country + controlNumber)
  //dividing this by 97 should have a remainder of 1
  val remainder = BigDecimal(check) % 97
  assert(remainder == 1)
  //return iban nr
  country + controlNumber + bank + accountPaddedTo10

}

createIBAN("ACME", "NL", "887373763") 
res0: String = NL14ACME0887373763
```

It works for all my bankaccount numbers although i've excluded them from this example for obvious reasons
the number here is randomly calculated
