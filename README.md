# node-nist

A simple low-level ANSI/NIST-ITL 1-2011 (update 2015) encoding and decoding utility library.
Written in Typescript for Node.

## Table of contents

- [node-nist](#node-nist)
  - [Table of contents](#table-of-contents)
  - [Installation](#installation)
  - [Usage](#usage)
    - [Encoding](#encoding)

## Installation

Install package

   ```bash
   npm install --save node-nist
   ```

## Usage

Encoding and decoding are described separately but they share many concepts.

In particular, `nistEncode` and `nistDecode` return a result (they do not throw exceptions).
A `Result` is a TypeScript discriminated union where a `Success` is tagged with `tag === 'success`
and a `Failure` with `tag === 'failure`.
In fact, `Result` is very similar to `Either` monad known from functional programming languages.

`nistEncode` and `nistDecode` could work on the NIST object alone. But they work best when codec
options are specific as their second argument. This way you can specify default values and impose
validation checks on individual NIST fields.

### Encoding

1. Encode a simple NIST object containing only Type-1 and Type-2 records:

```ts
import { nistEncode, NistFile } from 'node-nist';

const nist: NistFile = {
  1: {
    2: '0502', // version
    4: 'CRM', // TOT
    5: '20190717', // date
    7: 'DAI035454', // DAI
    8: 'ORI38574354', // ORI
    9: 'TCN2487S054' // TCN
  },
  2: {
    4: 'John',
    5: 'Doe',
    7: '1978-05-12'
  }
};

const encodeResult = nistEncode(nist, {});
if (encodeResult.tag === 'success') {
  const buffer = encodeResult.value;
  // perform action on successfull encode, such as sending out the buffer
} else {
  const error = encodeResult.error;
  // perform action on unsuccessfull encode, such as logging an error
}
```

2. Encode a NIST object with a different set of Type-2 record fields, containing also two
fingerprints:

```ts
import { nistEncode, NistFile } from 'node-nist';

const nist: NistFile = {
  1: {
    2: '0300', // version
    4: 'IDN', // TOT
    5: '20190722', // date
    7: 'ABIS', // DAI
    8: 'MID00001', // ORI
    9: 'SEN20190722093422-IDE-00040' // TCN
  },
  2: {
    901: 'MID00001'
  },
  4: [
    {
      3: '0', // flat finger
      4: ['7'], // left index finger
      9: 'fingerprint binary data'
    },
    {
      3: '0', // flat finger
      4: ['2'], // right index finger
      9: 'fingerprint binary data'
    }
  ]
};

const encodeResult = nistEncode(nist, {});
if (encodeResult.tag === 'success') {
  const buffer = encodeResult.value;
  // perform action on successfull encode, such as sending out the buffer
} else {
  const error = encodeResult.error;
  // perform action on unsuccessfull encode, such as logging an error
}
```

3. Encode a NIST object containing two fingerprints. Specify also NistEncodeOptions which
are used to automatically populate default values and to check whether the input NIST object
conforms to all the validation rules.

```ts
import { nistEncode, NistEncodeOptions, NistFile } from 'node-nist';

const nistEncodeOptions: NistEncodeOptions = {
  codecOptions: {
    MAP: { // TOT-specific encode options
      1: {
        6: {
          defaultValue: '5',
          mandatory: true,
          regexs: [{ regex: '^[1-9]{1}$', errMsg: 'Expected a number between 1 and 9' }]
        }
      }
    },
    default: {
      1: {
        2 /* ver */: {
          defaultValue: '0502', // conforming to ANSI/NIST-ITL 1-2011 edition 3 update 2015
          formatter: (field: NistField) => String(field.value).padStart(4, '0'),
          mandatory: true,
          maxLength: 4,
          minLength: 3,
          regexs: [{ regex: '^[0-9]{3,4}$', errMsg: 'Expected three or four digits' }]
        },
        4 /* tot */: {
          mandatory: true,
          maxLength: 16
        },
        5 /* dat */: {
          mandatory: true,
          regexs: [{ regex: '^[0-9]{8}$', errMsg: 'Expected eight digits' }]
        },
        6 /* pry */: {
          mandatory: false,
          regexs: [{ regex: '^[1-9]{1}$', errMsg: 'Expected a number between 1 and 9' }]
        },
        7 /* dai */: {
          mandatory: true
        },
        8 /* ori */: {
          mandatory: true
        },
        9 /* tcn */: {
          mandatory: true
        },
        10 /* tcr */: {
          mandatory: false
        },
        11 /* nsr */: {
          defaultValue: (field: NistField, nist: NistFile) => getFingerprintResolution(nist),
          mandatory: true,
          regexs: [{ regex: '^[0-9]{2}.[0-9]{2}$', errMsg: 'Expected a string in format dd.dd' }]
        },
        12 /* ntr */: {
          defaultValue: (field: NistField, nist: NistFile) => getFingerprintResolution(nist),
          mandatory: true,
          regexs: [{ regex: '^[0-9]{2}.[0-9]{2}$', errMsg: 'Expected a string in format dd.dd' }]
        }
      },
      4: {
        3 /* imp */: {
          mandatory: true
        },
        4 /* fgp */: {
          mandatory: true
        },
        5 /* isr */: {
          defaultValue: '0', // 500 ppi
          mandatory: true
        },
        6 /* hll */: {
          defaultValue: () => getHorizontalScanningLine(),
          mandatory: true
        },
        7 /* vll */: {
          defaultValue: () => getVerticalScanningLine(),
          mandatory: true
        },
        8 /* cga */: {
          defaultValue: '1', // WSQ
          mandatory: true
        }
      }
    }
  }
};

const nist: NistFile = {
  1: {
    2: '0300', // version
    4: 'MAP', // TOT
    5: '20190722', // date
    7: 'ABIS', // DAI
    8: 'MID00001', // ORI
    9: 'SEN20190722093422-IDE-00040' // TCN
  },
  2: {
    901: 'MID00001'
  },
  4: [
    {
      3: '0', // flat finger
      4: ['7'], // left index finger
      9: 'fingerprint binary data'
    },
    {
      3: '0', // flat finger
      4: ['2'], // right index finger
      9: 'fingerprint binary data'
    }
  ]
};

const encodeResult = nistEncode(nist, {});
if (encodeResult.tag === 'success') {
  const buffer = encodeResult.value;
  // perform action on successfull encode, such as sending out the buffer
} else {
  const error = encodeResult.error;
  // perform action on unsuccessfull encode, such as logging an error
}
```


