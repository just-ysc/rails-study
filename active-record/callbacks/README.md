# [Active Record Callbacks](https://guides.rubyonrails.org/active_record_callbacks.html)

## The Object Life Cycle

- 객체가 생성되고, 변경되고, 삭제되는 생명 주기 내에서, Active Record는 각 단계에서 사용할 수 있는 hook을 제공
    - 이 hook을 callback이라고 부

```ruby
class BirthdayCake < ApplicationRecord
  after_create -> { Rails.logger.info("Congratulations, the callback has run!") } 
end
```

## Callback Registration

- callback 구현 방법
  - method
    ```ruby
        class User < ApplicationRecord
          validates :username, :email, presence: true
          before_validation :ensure_username_has_value
    
          private
            def ensure_username_has_value
              if username.blank?
                self.username = email
              end
            end
        end
    ```
  - block
    ```ruby
        class User < ApplicationRecord
          validates :username, :email, presence: true
          before_validation do
             self.username = email if username.blank?
          end
        end
    ```
  - proc
    ```ruby
        class User < ApplicationRecord
          validates :username, :email, presence: true
          before_validation ->(user) { user.username = user.email if user.username.blank? }
        end        
    ```
  - class
    ```ruby
        class User < ApplicationRecord
          validates :username, :email, presence: true
          before_validation AddUsername
        end
    
        class AddUsername
          def self.before_validation(record)
            if record.username.blank?
              record.username = record.email
            end
          end
        end
    ```
  - module

### Registering Callbacks to Fire on Life Cycle Events

- `:on` 옵션을 사용하면 특정한 이벤트에만 callback이 실행되도록 할 수 있
```ruby
class User < ApplicationRecord
  validates :username, :email, presence: true
  
  # 생성 시에만 callback 실행
  before_validation :ensure_username_has_value, on: :create
  
  # 생성/변경 시에만 callback 실행
  after_validation :set_location, on: [:create, :update]
  
  private
    def ensure_username_has_value
      if username.blank?
        self.username = email
      end
    end
  
    def set_location
      self.location = LocationService.query(self)
    end
end
```

## Available Callbacks

- 생명 주기 이벤트별로 사용 가능한 callback 목록

### Creating an Object

- validation callbacks
  - `valid?`, `invalid?`와 같이 직접 validation을 실행시키거나, `create`, `update`, `save`와 같이 간접적으로 validation을 실행시킬 때
    - `before_validation`
    - `after_validation` 
- save callbacks
  - `create`, `update`, `save`를 통해 데이터베이스에 데이터를 영속화할 때
  - `before_save`
  - `around_save`
  - `after_save`
- create callbacks
  - `create`, `save`를 통해 데이터베이스에 처음으로 데이터를 영속화할 때
    - `before_create`
    - `around_create`
    - `after_create`
- `after_commit` / `after_rollback`

### Updating an Object

- `before_validation`
- `after_validatoin`
- `before_save`
- `around_save`
- `before_update`
- `around_update`
- `after_update`
- `after_save`
- `after_commit` / `after_rollback`

### Destroying an Object

- `before_destroy`
- `around_destroy`
- `after_destroy`
- `after_commit` / `after_rollback`

### `after_initialize` and `after_find`

- `after_initialize`: `new`로 새로운 객체를 생성하거나, 데이터베이스에서 객체를 로드했을 때
- `after_find`: 데이터베이스에서 객체를 로드했을 때

### `after_touch`

- `touch`: 객체의 타임스탬프 값을 현재 시간 혹은 원하는 시간 기준으로 변경하는 메소드
- `after_touch`: `touch`가 수행되고 나서 실행되는 callback
```ruby
class Book < ApplicationRecord
  belongs_to :library, touch: true
  after_touch do
    Rails.logger.info("A Book was touched")
  end
end

class Library < ApplicationRecord
  has_many :books
  after_touch :log_when_books_or_library_touched

  private
    def log_when_books_or_library_touched
      Rails.logger.info("Book/Library was touched")
    end
end
```

```shell
irb> book = Book.last
=> #<Book id: 1, library_id: 1, created_at: "2013-11-25 17:04:22", updated_at: "2013-11-25 17:05:05">

irb> book.touch # triggers book.library.touch
A Book was touched
Book/Library was touched
=> true
```

## Running Callbacks

- callback을 실행시키는 메소드는 다음과 같다.
  - `create`
  - `create!`
  - `destroy`
  - `destroy!`
  - `destroy_all`
  - `destroy_by`
  - `save`
  - `save!`
  - `save(validate: false)`
  - `save!(validate: false)`
  - `toggle!`
  - `touch`
  - `update_attribute`
  - `update_attribute!`
  - `update`
  - `update!`
  - `valid?`
  - `validate`
- 추가적으로, `after_find` callback은 다음의 finder 메소드에 의해서도 실행이 된다.
  - `all`
  - `first`
  - `find`
  - `find_by`
  - `find_by!`
  - `find_by_*`
  - `find_by_*!`
  - `find_by_sql`
  - `last`
  - `sole`
  - `take`

## Conditional Callbacks

- validations과 같이, `:if`, `:unless` 옵션을 통해 주어진 predicate을 만족할 경우에만 callback을 실행시킬 수 있다.
- `:if`, `:unless`를 동시에 사용할 수도 있다. 이 때, 두 옵션을 모두 만족해야지만 callback이 실행된다.

### Using `:if` and `:unless` with a `Symbol`

- `:if`, `:unless` 뒤에 메소드명을 symbol로 명시할 수 있다.
  - `:if`일 때 메소드가 `true`를 반환할 경우 callback 실행
  - `:unless`일 때 메소드가 `false`를 반환할 경우 callback 실

### Using `:if` and `:unless` with a `Proc`

```ruby
class Order < ApplicationRecord
  before_save :normalize_card_number,
    if: ->(order) { order.paid_with_card? }
end
# 간략하게 다음과 같이 작성하는 것도 가능하다
class Order < ApplicationRecord
  before_save :normalize_card_number, if: -> { paid_with_card? }
end
```