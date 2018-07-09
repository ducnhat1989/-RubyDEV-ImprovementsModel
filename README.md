# -RubyDEV-ImprovementsModel

Models:
1. [User](https://github.com/ducnhat1989/-RubyDEV-ImprovementsModel#improve-user-model)
Low: 2 - 
Medium: 2 - 
Critical: 2
2. [UserCourse](https://github.com/ducnhat1989/-RubyDEV-ImprovementsModel#improve-user-course-model)
Low: 1 - 
Critical: 1 -

### Improve User Model

#### Low
1. Group class method

When you have multiple class method, you should group them into one group, so it's easier to manage.

Refer: [RubyStyleGuide](https://github.com/rubocop-hq/ruby-style-guide#def-self-class-methods)

Currently, in the User model, the class method is not focused, so the optimization is as follows

```User.rb
.....
class << self
  def languages(local_lang = false)
    if local_lang
      Hash[%i[en ja].map{|lang| [lang, I18n.t(lang, scope: 'languages', locale: lang)] }]
    else
      Hash[%i[en ja].map{|lang| [lang, I18n.t(lang, scope: 'languages')] }]
    end
  end

  def generate_random_password(length = 12)
    RandomString.generate_random_string(length)
  end

  def common_keys
    keys = OnpremisePolicy.enabled? ? [:login] : [:email]
    keys = [:first_name_alphabet, :last_name_alphabet, :first_name_local, :last_name_local, :language]
    keys
  end

  def find_for_database_authentication(warden_conditions)
    conditions = warden_conditions.dup
    if login_or_email = conditions.delete(:login_or_email)
      where(conditions.to_h).with_login_or_email(login_or_email).first
    elsif conditions.has_key?(:login) || conditions.has_key?(:email)
      where(conditions.to_h).first
    end
  end

  def find_by_login_or_email_and_password(login_or_email, password)
    user = self.with_login_or_email(login_or_email).first
    return false unless user
    return false unless user.valid_password?(password)
    user
  end  
end
```

2. Improve instance method `common_keys`

```User.rb
def common_keys
  keys = OnpremisePolicy.enabled? ? [:login] : [:email]
  keys += [:first_name_alphabet, :last_name_alphabet, :first_name_local, :last_name_local, :language]
  keys
end
```
Assignment `+=` is not necessary because this case, the variable is a local variable, use of assignment there are no many meaning. We will correct as follow:

```User.rb
def common_keys
  keys = OnpremisePolicy.enabled? ? [:login] : [:email]
  keys + [:first_name_alphabet, :last_name_alphabet, :first_name_local, :last_name_local, :language]
end
```

#### Medium
1. Optimize scope `with_organization`

```User.rb
scope :with_organization, -> (organization) {
  if OnpremisePolicy.enabled?
    nil
  elsif organization
    if organization.class == Channel
      joins(:channels).where(organizations: {id: organization.id})
    else
      joins(:organizations).where(organizations: {id: organization.id})
    end
  end
}
```

The relationship of `user` and `channel` through `channel_users` so when joins `user` and `channel`, we will joins 3 tables (users - channel_users - channels).
In this case, we can just join 2 tables (user - channel_users) and still ensure the correct result because of channel_users also incluce the value of organization id.
Similar to organizations, we have the following new code:

```User.rb
scope :with_organization, -> (organization) {
  if OnpremisePolicy.enabled?
  elsif organization
    if organization.class == Channel
      joins(:channel_users).where(channel_users: {organization_id: organization.id})
    else
      joins(:organization_users).where(organization_users: {organization_id: organization.id})
    end
  end
}
```

This change will help our query reduce the number of tables in the joins sql, thereby speeding up the query.

2. Change `first` to `take`, omit order by clause

Vi du:

```User.rb
def find_for_database_authentication(warden_conditions)
  conditions = warden_conditions.dup
  if login_or_email = conditions.delete(:login_or_email)
    where(conditions.to_h).with_login_or_email(login_or_email).first
  elsif conditions.has_key?(:login) || conditions.has_key?(:email)
    where(conditions.to_h).first
  end
end
```

Notice that the queries generated from this method always have the `LIMIT 1 ASC` clause.
Using the order clause will slow our query speed as the SQL will execute the order before the limit clause. (Refer: [limit mysql](https://dev.mysql.com/doc/refman/5.7/en/limit-optimization.html))

In this case, the ordering is not required, so I suggest replacing `first` with `take` (Refer: [take or first](https://stackoverflow.com/questions/18496421/take-vs-first-performance-in-ruby-on-rails)). 

#### Critical

1. Optimize N+1 query of instance method `visible_user_ids`

```User.rb
if system_admin?
  User.pluck(:id)
else
  (organizations.map(&:users).flatten.map(&:id) + channels.map(&:users).flatten.map(&:id) + channels.map(&:organizations).flatten.map(&:users).flatten.map(&:id)).uniq
end
```

This method encountered a quite serious problem is N+1 query. You can refer to this error [here](https://secure.phabricator.com/book/phabcontrib/article/n_plus_one/). There are quite a few ways to fix this, here I will use `joins`.

```User.rb
if system_admin?
  User.pluck(:id)
else
  (
    organizations.joins(:users).pluck(:id) +
    channels.joins(:users).pluck(:id) +
    channels.joins(organizations: :organization_users).pluck(:user_id)
  ).uniq
end
```

It looks better, but can improve a bit. This code has the `+` array operation, which is quite resource consuming, so we can avoid the `+` array operator as follows:

```User.rb
if system_admin?
  User.pluck(:id)
else
  [
    *(organizations.joins(:users).pluck :id),
    *(channels.joins(:users).pluck :id),
    *(channels.joins(organizations: :organization_users).pluck :user_id)
  ].uniq
end
```

Alse, in this method, I see one more problem. Notice this method in models called by `authorized_with` scope of the User model.

```User.rb
scope :authorized_with, -> (user) {
  query = where(role: UserPolicy.available_role_types(user.role))
  if OnpremisePolicy.enabled?
    query
  elsif user.system_admin?
    query # includes/join here conflicts with that in 'with_organization'
  else
    query.where(id: user.visible_user_ids)
  end
}
```

You found the condition `user.system_admin?` is used in both `authorized_with` scope and `visible_user_ids` method, so this condition can be called 2 times, this is not necessary. However, be cautious when deleting conditions in `visible_user_ids` method because this method can be called by a service or controller. So make sure you do not alter the logic of the `visible_user_ids` method. How to edit as follows:

```User.rb
scope :authorized_with, -> (user) {
  query = where(role: UserPolicy.available_role_types(user.role))
  if OnpremisePolicy.enabled?
    query
  elsif user.system_admin?
    query # includes/join here conflicts with that in 'with_organization'
  else
    query.where(id: user.visible_user_ids_not_admin)
  end
}

def visible_user_ids
  if system_admin?
    User.pluck(:id)
  else
    visible_user_ids_not_admin
  end
end

private
  def visible_user_ids_not_admin
    [
      *(organizations.joins(:users).pluck :id),
      *(channels.joins(:users).pluck :id),
      *(channels.joins(organizations: :organization_users).pluck :user_id)
    ].uniq
  end

```

2. Optimize N+1 query of instance method `study_time_on_average`

```User.rb
def study_time_on_average
  return nil unless user_courses.any?
  study_trackings = user_courses.map(&:study_tracking).compact
  return nil unless study_trackings.any?
  study_trackings.map(&:study_time).sum / user_courses.size
end
```

Similar to the above case, we also see N+1 query here. And use `joins` is still good.

```User.rb
def study_time_on_average
  return nil unless user_courses.any?
  study_trackings = user_courses.joins(:study_tracking)
  return nil unless study_trackings.any?
  study_trackings.sum(:study_time) / user_courses.size
end
```

OK, now we have no more N+1 query, but you alse see that I have modified the last code line ?

Why I correcting this, there are 2 reasons for this:

- The old method, we must initialize array after use `sum` with the array, our app must initialize memory for the array, the larger the array, the larger the memory. While with the new method, we don't need to initialize array => processor speed will be improved, our app also less resource.
- This is a problem with the SUM array in this case. That is, if the `nil` element is present in the array, the `sum` method will issue a `NoMethodError: undefined method `+' for nil:NilClass`. However, if you use `SUM` with SQL then SQL solves this problem for us.

### Improve UserCourse Model

#### Low

1. Use `||=` operator instead of `if nil?`

```UserCourse.rb
def set_access_dates
  if program.fixed_dates?
    self.access_start_date = course.actual_access_start_date(false) if self.access_start_date.nil?
    self.access_end_date = course.actual_access_end_date(false) if self.access_end_date.nil?
  elsif program.rolling_start?
    today = Time.current.in_time_zone(timezone).to_date
    self.access_start_date = today if self.access_start_date.nil?
    self.access_end_date = access_end_date_for_rolling_start if self.access_end_date.nil?
  end
end
```

This code is quite long, if you use `||=` operator, you can write code shorter.

```UserCourse.rb
def set_access_dates
  if program.fixed_dates?
    self.access_start_date ||= course.actual_access_start_date(false)
    self.access_end_date ||= course.actual_access_end_date(false)
  elsif program.rolling_start?
    self.access_start_date ||= Time.current.in_time_zone(timezone).to_date
    self.access_end_date ||= access_end_date_for_rolling_start
  end
end
```

#### Critical

1. Issue of `with_course_progress` scope

```UserCourse.rb
scope :with_course_progress, -> (number, operator) {
  selected_ids = all.select { |uc| uc.course_progress.send(convert_operator_for_ruby(operator), number.to_i) }.map(&:id)
  where(id: selected_ids)
}
```

Looking at this scope, I immediately noticed `all.select{}`. Why loading all data ? I understand when read the code a bit, for example: `with_course_progress("gt", 40)` will return UserCourses with `course_progress` > 40.

As such, I understand that the code writer loads all the data because he does not know the value of `course_progress`. This approach is very bad, because if the amout of data of the UserCourse, then how this scope will handle? And I'm sure that it can not be processed quickly, here he moves all data into an array for processing, if the data is large, the array will grow, and this is not good with memory.

One way to improve this is to calculate the value of `course_progress` and then save it in a `course_progress` column in the UserCourse. And when you need it, just query in the `sourse_progress`.

Similarly, we can apply to improve the scope `with_progress_status`
