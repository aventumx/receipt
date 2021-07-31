![travisci](https://api.travis-ci.org/excid3/receipts.svg)

# Receipts

Receipts for your Rails application that works with any payment provider.

Check out the [example receipt](https://github.com/excid3/receipts/blob/master/examples/receipt.pdf?raw=true) and [example invoice](https://github.com/excid3/receipts/blob/master/examples/invoice.pdf?raw=true) PDFs.

## Installation

Add this line to your application's Gemfile:

```ruby
gem 'receipts'
```

And then execute:

    $ bundle

Or install it yourself as:

    $ gem install receipts

## Usage

Adding receipts to your application is pretty simple. All you need is a
model that stores your transaction details. In this example our
application has a model named `Charge` that we will use.

We're going to add a method called `receipt` on our model called `Charge`
that will create a new receipt for the charge using attributes from the
model.

Video Tutorial:
[GoRails Episode #51](https://gorails.com/episodes/pdf-receipts)

```ruby
# == Schema Information
#
# Table name: charges
#
#  id             :integer          not null, primary key
#  user_id        :integer
#  stripe_id      :string(255)
#  amount         :integer
#  card_last4     :string(255)
#  card_type      :string(255)
#  card_exp_month :integer
#  card_exp_year  :integer
#  uuid           :string
#  created_at     :datetime
#  updated_at     :datetime
#

class Charge < ActiveRecord::Base
  belongs_to :user
  validates :stripe_id, uniqueness: true

  def receipt
    Receipts::Receipt.new(
      id: id,
      subheading: "RECEIPT FOR CHARGE #%{id}",
      product: "GoRails",
      company: {
        name: "GoRails, LLC.",
        address: "123 Fake Street\nNew York City, NY 10012",
        email: "support@example.com",
        logo: Rails.root.join("app/assets/images/logo.png")
      },
      line_items: [
        ["Date",           created_at.to_s],
        ["Account Billed", "#{user.name} (#{user.email})"],
        ["Product",        "GoRails"],
        ["Amount",         "$#{amount / 100}.00"],
        ["Charged to",     "#{card_type} (**** **** **** #{card_last4})"],
        ["Transaction ID", uuid]
      ],
      font: {
        bold: Rails.root.join('app/assets/fonts/tradegothic/TradeGothic-Bold.ttf'),
        normal: Rails.root.join('app/assets/fonts/tradegothic/TradeGothic.ttf'),
      }
    )
  end
end
```

Update the options for the receipt according to the data you want to
display.

## Customizing Your Receipts

* `id` - **Required**

This sets the ID of the charge on the receipt

* `product` or `message` - **Required**

You can set either the product or message options. If you set product, it will use the default message. If you want a custom message, you can set the message option to populate it with custom text.

* `company` - **Required**

Company consists of several required nested attributes.

  * `name` - **Required**
  * `address` - **Required**
  * `email` - **Required**
  * `line_items` - **Required**

You can set as many line items on the receipts as you want. Just pass in an array with each item containing a name and a value to display on the receipt.

  * `logo` - *Optional*

The logo must be either a string path to a file or a file-like object.

```ruby
logo: Rails.root.join("app/assets/images/logo.png")
# or
logo: File.open("app/assets/images/logo.png", "rb")
```

To use an image from a URL, we recommend using `open-uri` to open the remote file as a StringIO object.

`require 'open-uri'`

`logo: URI.open("https://www.ruby-lang.org/images/header-ruby-logo@2x.png")`

* `font` - *Optional*

If you'd like to use your own custom font, you can pass in the file paths to the `normal` and `bold` variations of your font. The bold font variation is required because it is used in the default message. If you wish to override that, you can pass in your own custom message instead.


## Rendering the Receipt PDF in your Controller

Here we have a charges controller that responds to the show action. When
you visit it with the PDF format, it calls the `receipt` method that we
just created on the `Charge` model.

We set the filename to be the date plus the product name. You can
customize the filename to your liking.

Next we set the response type which will be `application/pdf`

Optionally we can set the `disposition` to `:inline` which allows us to
render the PDF in the browser without forcing the download. If you
delete this option or change it to `:attachment` then the receipt will
be downloaded instead.

```ruby
class ChargesController < ApplicationController
  before_action :authenticate_user!
  before_action :set_charge

  def show
    respond_to do |format|
      format.pdf {
        send_data @charge.receipt.render,
          filename: "#{@charge.created_at.strftime("%Y-%m-%d")}-gorails-receipt.pdf",
          type: "application/pdf",
          disposition: :inline
      }
    end
  end

  private

    def set_charge
      @charge = current_user.charges.find(params[:id])
    end
end
```

And that's it! Just create a `link_to` to your charge with the format of
`pdf` and you're good to go.

For example:

```ruby
# config/routes.rb
resources :charges
```

```erb
<%= link_to "Download Receipt", charge_path(@charge, format: :pdf) %>
```

## Invoices

Invoices follow the exact same set of steps as above, with a few minor changes and have a few extra arguments you can use:

* `issue_date` - Date the invoice was issued

* `due_date` - Date the invoice payment is due

* `status` - A status for the invoice (Pending, Paid, etc)

* `bill_to` - A string or Array of lines with billing details

You can also use line_items to flexibly generate and display the table with items in it, including subtotal, taxes, and total amount.

```ruby
  Receipts::Invoice.new(
    id: "123",
    issue_date: Date.today,
    due_date: Date.today + 30,
    status: "<b><color rgb='#5eba7d'>PAID</color></b>",
    bill_to: [
      "GoRails, LLC",
      "123 Fake Street",
      "New York City, NY 10012",
      nil,
      "mail@example.com",
    ],
    company: {
      name: "GoRails, LLC",
      address: "123 Fake Street\nNew York City, NY 10012",
      email: "support@example.com",
      logo: File.expand_path("./examples/gorails.png")
    },
    line_items: [
      ["<b>Item</b>", "<b>Unit Cost</b>", "<b>Quantity</b>", "<b>Amount</b>"],
      ["GoRails Subscription", "$19.00", "1", "$19.00"],
      [nil, nil, "Subtotal", "$19.00"],
      [nil, nil, "Tax Rate", "0%"],
      [nil, nil, "Total", "$19.00"],
    ],
  )
```

## Statements

Statements follow the exact same set of steps as receipts, with a few minor changes and have a few extra arguments you can use:

* `issue_date` - Date the invoice was issued

* `start_date` - The start date of the statement period

* `start_date` - The end date of the statement period

* `bill_to` - A string or Array of lines with account details

You can also use line_items to flexibly generate and display the table with items in it, including subtotal, taxes, and total amount.

```ruby
  Receipts::Statement.new(
    id: "123",
    issue_date: Date.today,
    start_date: Date.today - 30,
    end_date: Date.today,
    bill_to: [
      "GoRails, LLC",
      "123 Fake Street",
      "New York City, NY 10012",
      nil,
      "mail@example.com",
    ],
    company: {
      name: "GoRails, LLC",
      address: "123 Fake Street\nNew York City, NY 10012",
      email: "support@example.com",
      logo: File.expand_path("./examples/gorails.png")
    },
    line_items: [
      ["<b>Item</b>", "<b>Unit Cost</b>", "<b>Quantity</b>", "<b>Amount</b>"],
      ["GoRails Subscription", "$19.00", "1", "$19.00"],
      [nil, nil, "Subtotal", "$19.00"],
      [nil, nil, "Tax Rate", "0%"],
      [nil, nil, "Total", "$19.00"],
    ],
  )
```
## Internationalization (I18n)
 You can also use subheading, bill_to_text, issue_date_text, due_date_text, message to modify default text to your need.
```ruby

Receipts::Invoice.new(
      id: id,
      subheading: "#{I18n.t('invoice_id')} ##{invoice_number}",
      bill_to_text: I18n.t('bill_to').upcase,
      issue_date_text: I18n.t('issue_date').upcase,
      due_date_text: I18n.t('renewal').upcase,
      status_text: I18n.t('state').upcase,
      message: I18n.t('invoice_pdf_message', id: id, email: 'support@mail.com').capitalize,
      issue_date: active_from.strftime('%F'),
      due_date: active_till.strftime('%F'),
      status: "<b><color rgb='##{badgeColor}'>#{badgeLabel}</color></b>",
    )
```
## Contributing

1. Fork it ( https://github.com/excid3/receipts/fork )
2. Create your feature branch (`git checkout -b my-new-feature`)
3. Commit your changes (`git commit -am 'Add some feature'`)
4. Push to the branch (`git push origin my-new-feature`)
5. Create a new Pull Request

