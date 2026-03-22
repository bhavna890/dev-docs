# Show percentage off on discount badge on Shopify (Free theme)

## `card-product.liquid or product-card.liquid`

        ```html
        {{card_product.compare_at_price | minus: card_product.price | times: 100 | divided_by: card_product.compare_at_price }}% OFF
        ```

## `price.liquid`

        ```html
        {{compare_at_price | minus: price | times: 100 | divided_by:  compare_at_price }}% OFF
        ```