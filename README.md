# substrate-course

### substrate

在pallet下新建一个rust包

```shell
 cargo new poe --lib
```

更新Cargo.toml，声明一些需要依赖的包

```toml
[package]
name = "poe"
version = "0.1.0"
authors = ["Aaron <armatrix@163.com>"]
edition = "2018"

# See more keys and their definitions at https://doc.rust-lang.org/cargo/reference/manifest.html

[package.metadata.docs.rs]
targets = ['x86_64-unknown-linux-gnu']

# alias "parity-scale-code" to "codec"
[dependencies.codec]
default-features = false
features = ['derive']
package = 'parity-scale-codec'
version = '1.3.4'

[dependencies]
frame-support = { default-features = false, version = '2.0.0' }
frame-system = { default-features = false, version = '2.0.0' }
sp-std = { default-features = false, version = '2.0.0' }

[dev-dependencies]
sp-core = { default-features = false, version = '2.0.0' }
sp-io = { default-features = false, version = '2.0.0' }
sp-runtime = { default-features = false, version = '2.0.0' }

[features]
default = ['std']
std = [
    'codec/std',
    'frame-support/std',
    'frame-system/std',
    'sp-std/std',  
]
```

代码逻辑 lib.rc

```rust
#![cfg_attr(not(feature = "std"), no_std)]

use frame_support::{
    decl_module, decl_storage, decl_event, decl_error, ensure, StorageMap
};
use frame_system::ensure_signed;
use sp_std::vec::Vec;

/// Configure the pallet by specifying the parameters and types on which it depends.
pub trait Trait: frame_system::Trait {
    /// Because this pallet emits events, it depends on the runtime's definition of an event.
    type Event: From<Event<Self>> + Into<<Self as frame_system::Trait>::Event>;
}

// Pallets use events to inform users when important changes are made.
// Event documentation should end with an array that provides descriptive names for parameters.
// https://substrate.dev/docs/en/knowledgebase/runtime/events
decl_event! {
    pub enum Event<T> where AccountId = <T as frame_system::Trait>::AccountId {
        /// Event emitted when a proof has been claimed. [who, claim]
        ClaimCreated(AccountId, Vec<u8>),
        /// Event emitted when a claim is revoked by the owner. [who, claim]
        ClaimRevoked(AccountId, Vec<u8>),
    }
}

// Errors inform users that something went wrong.
decl_error! {
    pub enum Error for Module<T: Trait> {
        /// The proof has already been claimed.
        ProofAlreadyClaimed,
        /// The proof does not exist, so it cannot be revoked.
        NoSuchProof,
        /// The proof is claimed by another account, so caller can't revoke it.
        NotProofOwner,
    }
}

// The pallet's runtime storage items.
// https://substrate.dev/docs/en/knowledgebase/runtime/storage
decl_storage! {
    trait Store for Module<T: Trait> as PoeModule {
        /// The storage item for our proofs.
        /// It maps a proof to the user who made the claim and when they made it.
        Proofs: map hasher(blake2_128_concat) Vec<u8> => (T::AccountId, T::BlockNumber);
    }
}

// Dispatchable functions allows users to interact with the pallet and invoke state changes.
// These functions materialize as "extrinsics", which are often compared to transactions.
// Dispatchable functions must be annotated with a weight and must return a DispatchResult.
decl_module! {
    pub struct Module<T: Trait> for enum Call where origin: T::Origin {
        // Errors must be initialized if they are used by the pallet.
        type Error = Error<T>;

        // Events must be initialized if they are used by the pallet.
        fn deposit_event() = default;

        /// Allow a user to claim ownership of an unclaimed proof.
        #[weight = 10_000]
        fn create_claim(origin, proof: Vec<u8>) {
            // Check that the extrinsic was signed and get the signer.
            // This function will return an error if the extrinsic is not signed.
            // https://substrate.dev/docs/en/knowledgebase/runtime/origin
            let sender = ensure_signed(origin)?;

            // Verify that the specified proof has not already been claimed.
            ensure!(!Proofs::<T>::contains_key(&proof), Error::<T>::ProofAlreadyClaimed);

            // Get the block number from the FRAME System module.
            let current_block = <frame_system::Module<T>>::block_number();

            // Store the proof with the sender and block number.
            Proofs::<T>::insert(&proof, (&sender, current_block));

            // Emit an event that the claim was created.
            Self::deposit_event(RawEvent::ClaimCreated(sender, proof));
        }

        /// Allow the owner to revoke their claim.
        #[weight = 10_000]
        fn revoke_claim(origin, proof: Vec<u8>) {
            // Check that the extrinsic was signed and get the signer.
            // This function will return an error if the extrinsic is not signed.
            // https://substrate.dev/docs/en/knowledgebase/runtime/origin
            let sender = ensure_signed(origin)?;

            // Verify that the specified proof has been claimed.
            ensure!(Proofs::<T>::contains_key(&proof), Error::<T>::NoSuchProof);

            // Get owner of the claim.
            let (owner, _) = Proofs::<T>::get(&proof);

            // Verify that sender of the current call is the claim owner.
            ensure!(sender == owner, Error::<T>::NotProofOwner);

            // Remove claim from storage.
            Proofs::<T>::remove(&proof);

            // Emit an event that the claim was erased.
            Self::deposit_event(RawEvent::ClaimRevoked(sender, proof));
        }
    }
}
```

在runtime中添加

cargo.toml

```toml
[dependencies]
# --snip--

# local dependencies
pallet-template = { path = '../pallets/template', default-features = false, version = '2.0.0' }
# 添加该依赖
poe = { path = '../pallets/poe', default-features = false, version = '0.1.0' }

[features]
# --snip--
std = [
    # --snip--
    'sp-version/std',
    # 添加该feature
    'poe/std'
]
```

lib.rs

```rust
// 导入该pallet
pub use poe;

// 为runtime实现该trait
impl poe::Trait for Runtime {
	type Event = Event;
}

// 创建运行时的pallet
construct_runtime!(
	pub enum Runtime where
		Block = Block,
		NodeBlock = opaque::Block,
		UncheckedExtrinsic = UncheckedExtrinsic
	{
		// --snip--
		// Include the custom logic from the template pallet in the runtime.
		TemplateModule: pallet_template::{Module, Call, Storage, Event<T>},
    // 添加该依赖
		PoeModule: poe::{Module, Call, Storage, Event<T>},
	}
);
```

### 前端

新建一个PoeModule.js

```react
// React and Semantic UI 元素。
import React, { useState, useEffect } from 'react';
import { Form, Input, Grid, Message } from 'semantic-ui-react';
// 预设的 Substrate front-end 工具，用于连接节点和执行交易。
import { useSubstrate } from './substrate-lib';
import { TxButton } from './substrate-lib/components';
// Polkadot-JS 工具，用作对数据进行 hash 计算 。
import { blake2AsHex } from '@polkadot/util-crypto';

// 导出存证的主要组件。
export function Main (props) {
  // 建立一个与 Substrate 节点通讯的 API。
  const { api } = useSubstrate();
  // 从 `AccountSelector` 组件中获取选定的用户。
  const { accountPair } = props;
  // React hooks 用于追踪所有变量的状态。
  // 更多内容参见：https://reactjs.org/docs/hooks-intro.html
  const [status, setStatus] = useState('');
  const [digest, setDigest] = useState('');
  const [owner, setOwner] = useState('');
  const [block, setBlock] = useState(0);

  // 被后续代码中的函数所访问的 `FileReader()` 实例。
  let fileReader;

  // 使用 Blake2 256 散列函数生成文件的摘要。
  const bufferToDigest = () => {
    // 将文件内容转换成 Hex 格式。
    const content = Array.from(new Uint8Array(fileReader.result))
      .map((b) => b.toString(16).padStart(2, '0'))
      .join('');

    const hash = blake2AsHex(content, 256);
    setDigest(hash);
  };

  // 当一个新文件被选取时需调用的回调函数。
  const handleFileChosen = (file) => {
    fileReader = new FileReader();
    fileReader.onloadend = bufferToDigest;
    fileReader.readAsArrayBuffer(file);
  };

  // 使用 React hook 来更新文件中的 owner 和  block number 信息。
  useEffect(() => {
    let unsubscribe;

    // 使用 Polkadot-JS API 来查询 pallet 中  `proofs` 的存储信息。
    // pub-sub，订阅获取最新值。
    api.query.poeModule
      .proofs(digest, (result) => {
        // 我们的存储项将返回一个元组，以一个数组来代表。
        setOwner(result[0].toString());
        setBlock(result[1].toNumber());
      })
      .then((unsub) => {
        unsubscribe = unsub;
      });

    return () => unsubscribe && unsubscribe();
    // 用来告诉 React hook 在文件的摘要发生更改，刷夜页面
  }, [digest, api.query.poeModule]);

  // 若储存文件摘要的区块值不为零，我们则称该文件摘要已被声明。
  function isClaimed () {
    return block !== 0;
  }

  // 从组件中返回实际的 UI 元素。
  return (
    <Grid.Column>
      <h1>Proof Of Existence</h1>
      {/* 当文件被声明或失败时，显示警告或成功的消息。 */}
      <Form success={!!digest && !isClaimed()} warning={isClaimed()}>
        <Form.Field>
          {/* 回调函数为 `handleFileChosen` 的文件选择器。 */}
          <Input
            type='file'
            id='file'
            label='Your File'
            onChange={ e => handleFileChosen(e.target.files[0]) }
          />
          {/* 如果要声明的文件可用，则显示此消息 */}
          <Message success header='File Digest Unclaimed' content={digest} />
          {/* 如果文件已被声明，则显示此消息。 */}
          <Message
            warning
            header='File Digest Claimed'
            list={[digest, `Owner: ${owner}`, `Block: ${block}`]}
          />
        </Form.Field>
        {/* 与组件交互的 Buttons。 */}
        <Form.Field>
          {/* 用于创建声明的 Button。 只有在文件被选择且所选文件未被声明时，
          可用。 更新 `status`。 */}
          <TxButton
            accountPair={accountPair}
            label={'Create Claim'}
            setStatus={setStatus}
            type='SIGNED-TX'
            disabled={isClaimed() || !digest}
            attrs={{
              palletRpc: 'poeModule',
              callable: 'createClaim',
              inputParams: [digest],
              paramFields: [true]
            }}
          />
          {/* 用于撤销声明的 Button。 只有在文件被选择且所选文件已被声明时，
          可用。 更新 `status`。 */}
          <TxButton
            accountPair={accountPair}
            label='Revoke Claim'
            setStatus={setStatus}
            type='SIGNED-TX'
            disabled={!isClaimed() || owner !== accountPair.address}
            attrs={{
              palletRpc: 'poeModule',
              callable: 'revokeClaim',
              inputParams: [digest],
              paramFields: [true]
            }}
          />
        </Form.Field>
        {/* 交易的状态信息。 */}
        <div style={{ overflowWrap: 'break-word' }}>{status}</div>
      </Form>
    </Grid.Column>
  );
}

export default function PoeModule (props) {
  const { api } = useSubstrate();
  return (api.query.poeModule && api.query.poeModule.proofs
    ? <Main {...props} /> : null);
}

```

在App.js中引入该文件

```react
import React, { useState, createRef } from 'react';
import { Container, Dimmer, Loader, Grid, Sticky, Message } from 'semantic-ui-react';
import 'semantic-ui-css/semantic.min.css';

import { SubstrateContextProvider, useSubstrate } from './substrate-lib';
import { DeveloperConsole } from './substrate-lib/components';

import AccountSelector from './AccountSelector';
import Balances from './Balances';
import BlockNumber from './BlockNumber';
import Events from './Events';
import Interactor from './Interactor';
import Metadata from './Metadata';
import NodeInfo from './NodeInfo';
import TemplateModule from './TemplateModule';
import Transfer from './Transfer';
import Upgrade from './Upgrade';
// 导包
import PoeModule from './PoeModule';

function Main () {
  const [accountAddress, setAccountAddress] = useState(null);
  const { apiState, keyring, keyringState, apiError } = useSubstrate();
  const accountPair =
    accountAddress &&
    keyringState === 'READY' &&
    keyring.getPair(accountAddress);

  const loader = text =>
    <Dimmer active>
      <Loader size='small'>{text}</Loader>
    </Dimmer>;

  const message = err =>
    <Grid centered columns={2} padded>
      <Grid.Column>
        <Message negative compact floating
          header='Error Connecting to Substrate'
          content={`${err}`}
        />
      </Grid.Column>
    </Grid>;

  if (apiState === 'ERROR') return message(apiError);
  else if (apiState !== 'READY') return loader('Connecting to Substrate');

  if (keyringState !== 'READY') {
    return loader('Loading accounts (please review any extension\'s authorization)');
  }

  const contextRef = createRef();

  return (
    <div ref={contextRef}>
      <Sticky context={contextRef}>
        <AccountSelector setAccountAddress={setAccountAddress} />
      </Sticky>
      <Container>
        <Grid stackable columns='equal'>
          <Grid.Row stretched>
            <NodeInfo />
            <Metadata />
            <BlockNumber />
            <BlockNumber finalized />
          </Grid.Row>
          <Grid.Row stretched>
            <Balances />
          </Grid.Row>
          <Grid.Row>
            <Transfer accountPair={accountPair} />
            <Upgrade accountPair={accountPair} />
          </Grid.Row>
          <Grid.Row>
            <Interactor accountPair={accountPair} />
            <Events />
          </Grid.Row>
          <Grid.Row>
            <TemplateModule accountPair={accountPair} />
          </Grid.Row>
          <!-- 在原有的grid布局上新加入一个poe的 grid -->
          <Grid.Row>
            <PoeModule accountPair={accountPair} />
          </Grid.Row>
        </Grid>
      </Container>
      <DeveloperConsole />
    </div>
  );
}

export default function App () {
  return (
    <SubstrateContextProvider>
      <Main />
    </SubstrateContextProvider>
  );
}

```

