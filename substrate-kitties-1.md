# Substrate Kitties (1)

## 课程大纲 Substrate Kitties教程

- Metadata 元数据介绍
- 模块功能开发
- 单元测试
- FRAME资产相关模块介绍
    - balances
    - assets
- 作业

## 作业

编程作业，需要完成以下要求并且提交代码链接：

#### 1.增加买和卖的extrinsic，对kitties的实现进行重构，提取出公共代码

```
/// 出售 Kitty
		/// @dev: price 为 None 时, 表示取消出售
		#[pallet::weight(0)]
		pub fn sell(
			origin: OriginFor<T>,
			kitty_id: T::KittyIndex,
			price: Option<BalanceOf<T>>,
		) -> DispatchResult {
			let who = ensure_signed(origin)?;
			ensure!(Some(who.clone()) == Self::owner(kitty_id), Error::<T>::NotOwnerOfKitty);

			KittiesPrice::<T>::mutate_exists(kitty_id, |p| *p = Some(price));

			match price {
				Some(_) => {
					Self::deposit_event(Event::KittyForSale(who, kitty_id, price));
				}
				None => {
					Self::deposit_event(Event::KittyCancelSale(who, kitty_id));
				}
			}

			Ok(())
		}

		/// 购买 Kitty
		#[pallet::weight(0)]
		pub fn buy(origin: OriginFor<T>, kitty_id: T::KittyIndex) -> DispatchResult {
			let buyer = ensure_signed(origin)?;

			let owner = Self::owner(kitty_id).unwrap();
			ensure!(owner != buyer.clone(), Error::<T>::KittyAlreadyOwned);

			let price = Self::kitties_price(kitty_id).ok_or(Error::<T>::NotForSale)?;

			let reserve = T::ReserveOfNewCreate::get();

			// 扣除质押金额
			T::Currency::reserve(&buyer, reserve).map_err(|_| Error::<T>::NotEnoughBalance)?;

			// 出售方解除质押
			T::Currency::unreserve(&owner, reserve);

			// 转账
			T::Currency::transfer(
				&buyer,
				&owner,
				price,
				frame_support::traits::ExistenceRequirement::KeepAlive,
			)?;

			// 出售下架
			KittiesPrice::<T>::remove(kitty_id);

			Self::transfer_kitty(owner, buyer, kitty_id);

			Ok(())
		}
```

#### 2.kittyindex不在pallet中指定，而是在runtime里面绑定

```
impl pallet_kitties::Config for Runtime {
	type Event = Event;
	type Randomness = RandomnessCollectiveFlip;
	type KittyIndex = u32;
	type ReserveOfNewCreate = ReserveOfNewCreate;
	type Currency = Balances;
}

```

#### 3.测试代码能测试所有的五个方法，能检查所有的定义event，能测试出所有定义的错误类型

```

/// create new kitty
fn new_kitty(account_id: u64) -> DispatchResult {
	Kitties::create(Origin::signed(account_id))
}

#[test]
fn create_with_max_count_overflow() {
	new_test_ext().execute_with(|| {
		KittiesCount::<Test>::put(u32::max_value());
		assert_noop!(new_kitty(1), Error::<Test>::KittiesCountOverflow);
	});
}

#[test]
fn create_test_success_with_event() {
	new_test_ext().execute_with(|| {
		assert_ok!(new_kitty(1));
		assert_eq!(KittiesCount::<Test>::get(), Some(1));
		assert_event!(Event::KittyCreated(1, 1));
	});
}

#[test]
fn create_last_with_id_max_value() {
	new_test_ext().execute_with(|| {
		KittiesCount::<Test>::put(u32::max_value() - 1);
		assert_ok!(new_kitty(1));
		assert_eq!(KittiesCount::<Test>::get(), Some(u32::max_value()));
	});
}

#[test]
fn create_failed_with_not_enough_balance() {
	new_test_ext().execute_with(|| {
		assert_noop!(new_kitty(3), Error::<Test>::NotEnoughBalance);
	});
}

#[test]
fn transfer_success() {
	new_test_ext().execute_with(|| {
		let _ = new_kitty(1);
		assert_ok!(Kitties::transfer(Origin::signed(1), 2, 1));
		assert_event!(Event::KittyTransfered(1, 2, 1));
	});
}

#[test]
fn transfer_fail_when_to_some_owner() {
	new_test_ext().execute_with(|| {
		let _ = new_kitty(1);
		assert_noop!(Kitties::transfer(Origin::signed(1), 1, 1), Error::<Test>::SameOwner);
	});
}

#[test]
fn transfer_fail_not_owner() {
	new_test_ext().execute_with(|| {
		let _ = new_kitty(1);
		assert_noop!(Kitties::transfer(Origin::signed(2), 1, 1), Error::<Test>::NotOwnerOfKitty);
	});
}

#[test]
fn breed_success() {
	new_test_ext().execute_with(|| {
		let _ = new_kitty(1);
		let _ = new_kitty(1);

		assert_ok!(Kitties::breed(Origin::signed(1), 1, 2));
		assert_eq!(KittiesCount::<Test>::get(), Some(3));
		assert_event!(Event::KittyCreated(1, 3));
	});
}

#[test]
fn breed_fail_with_same_kitty_id() {
	new_test_ext().execute_with(|| {
		assert_noop!(Kitties::breed(Origin::signed(1), 1, 1), Error::<Test>::SameParentIndex);
	});
}

#[test]
fn breed_fail_with_invalid_index() {
	new_test_ext().execute_with(|| {
		assert_noop!(Kitties::breed(Origin::signed(1), 1, 2), Error::<Test>::InvalidKittyIndex);
	});
}

#[test]
fn breed_fail_with_invalid_owner() {
	new_test_ext().execute_with(|| {
		let _ = new_kitty(1);
		let _ = new_kitty(2);

		assert_noop!(Kitties::breed(Origin::signed(1), 1, 2), Error::<Test>::NotOwnerOfKitty);
	});
}

#[test]
fn breed_fail_with_count_overflow() {
	new_test_ext().execute_with(|| {
		let _ = new_kitty(1);
		let _ = new_kitty(1);

		KittiesCount::<Test>::put(u32::max_value());

		assert_noop!(Kitties::breed(Origin::signed(1), 1, 2), Error::<Test>::KittiesCountOverflow);
	});
}

#[test]
fn sell_fail_with_not_owner() {
	new_test_ext().execute_with(|| {
		assert_noop!(
			Kitties::sell(Origin::signed(1), 1, Some(100)),
			Error::<Test>::NotOwnerOfKitty
		);
	});
}

#[test]
fn sell_success() {
	new_test_ext().execute_with(|| {
		let _ = new_kitty(1);
		assert_ok!(Kitties::sell(Origin::signed(1), 1, Some(100)));
		assert_event!(Event::KittyForSale(1, 1, Some(100)));
	});
}

#[test]
fn cancel_sell_with_none_price() {
	new_test_ext().execute_with(|| {
		let _ = new_kitty(1);
		let _ = Kitties::sell(Origin::signed(1), 1, Some(100));

		assert_ok!(Kitties::sell(Origin::signed(1), 1, None));

		assert_eq!(None, KittiesPrice::<Test>::get(1));
		assert_event!(Event::KittyCancelSale(1, 1));
	});
}

#[test]
fn buy_failed_when_already_owned() {
	new_test_ext().execute_with(|| {
		let _ = new_kitty(1);
		assert_noop!(Kitties::buy(Origin::signed(1), 1), Error::<Test>::KittyAlreadyOwned);
	});
}

#[test]
fn buy_fail_when_not_for_sale() {
	new_test_ext().execute_with(|| {
		let _ = new_kitty(1);
		assert_noop!(Kitties::buy(Origin::signed(2), 1), Error::<Test>::NotForSale);
	});
}

#[test]
fn buy_fail_with_not_enough_balance() {
	new_test_ext().execute_with(|| {
		let _ = new_kitty(1);
		let _ = Kitties::sell(Origin::signed(1), 1, Some(100));

		assert_noop!(Kitties::buy(Origin::signed(3), 1), Error::<Test>::NotEnoughBalance);
	});
}

#[test]
fn buy_success() {
	new_test_ext().execute_with(|| {
		let _ = new_kitty(1);
		let _ = Kitties::sell(Origin::signed(1), 1, Some(100));

		assert_ok!(Kitties::buy(Origin::signed(2), 1));

		assert_eq!(KittiesPrice::<Test>::contains_key(1), false);

		assert_event!(Event::KittyTransfered(1, 2, 1));
	});
}
```

#### 4.引入balances里面的方法，在创建时质押一定数量的token，在购买时支付token

```
[dependencies.pallet-balances]
default-features = false
git = 'https://github.com/paritytech/substrate.git'
tag = 'monthly-2021-08'
version = '4.0.0-dev'
```

## 课程内容

- Metadata 元数据介绍

- Kitties Pallet开发

- Frame资产相关模块

    - Ballances

    - Assets

## Metadata 元数据

其中包含了每个模块的元数据

这里的使用场景是用来描述runtime模块

```
Storage
Events
Calls
Constants
Errors
```

Index 区块中函数的调用有很大关系

动态升级，不同的区块高度的区块中的metadata是不一样的

source code: 

    Substrate frame metadata lib.rs
    Substrate frame supprot metadata.rs

## Kitties Pallet

创建小猫

繁殖小猫

转移小猫

买卖小猫

显示小猫

## Balance

存储token 数量

账户拥有的数量

transfer

锁定资产

查询资产

balance对单一资产管理的模块

管理多个资产需要多次实例化balance

```
Existential Deposit: The minimum balance required to create or keep an account open. This prevents
"dust accounts" from filling storage. When the free plus the reserved balance (i.e. the total balance)
fall below this, then the account is said to be dead; and it loses its functionality as well as any
prior history and all information on it is removed from the chain's state.
No account should ever have a total balance that is strictly between 0 and the existential
deposit (exclusive). If this ever happens, it indicates either a bug in this module or an
erroneous raw mutation of storage.
```

```
Reserved Balance
```

```
Lock 多次lock重复使用
```

```
Existential Deposit
```










