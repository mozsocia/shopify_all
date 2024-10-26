Here's a detailed example of the folder structure and code for organizing theme-specific styles:

```
extension/
  ├── blocks/
  │   └── featured_product.liquid      # Your main block template
  ├── snippets/
  │   ├── block_styles.liquid         # Main style switcher
  │   ├── wokiee_styles.liquid        # Wokiee theme styles
  │   ├── ella_styles.liquid          # Ella theme styles
  │   └── default_styles.liquid       # Default fallback styles
  └── assets/
      └── block.css                   # Common/shared styles
```

1. Main Block Template (`blocks/featured_product.liquid`):
```liquid
{% render 'block_styles' %}

<div class="featured-product-block">
  <h2 class="product-title">{{ product.title }}</h2>
  <div class="product-price">{{ product.price | money }}</div>
  <button class="add-to-cart-button">Add to Cart</button>
</div>
```

2. Style Switcher (`snippets/block_styles.liquid`):
```liquid
<style>
  {% case request.design_theme.name %}
    {% when 'Wokiee' %}
      {% render 'wokiee_styles' %}
    {% when 'Ella' %}
      {% render 'ella_styles' %}
    {% else %}
      {% render 'default_styles' %}
  {% endcase %}
</style>
```

3. Theme-Specific Styles (`snippets/wokiee_styles.liquid`):
```liquid
.featured-product-block {
  padding: 20px;
  background: {{ settings.colors_background_1 }};
  border-radius: 8px;
}

.product-title {
  font-family: {{ settings.type_header_font_family }};
  font-size: 24px;
  color: {{ settings.colors_text }};
}

.add-to-cart-button {
  background: {{ settings.colors_accent_1 }};
  color: white;
  padding: 12px 24px;
  border-radius: 4px;
  border: none;
}
```

4. Another Theme Style (`snippets/ella_styles.liquid`):
```liquid
.featured-product-block {
  padding: 30px;
  background: {{ settings.colors_background_1 }};
  border: 1px solid {{ settings.colors_accent_2 }};
}

.product-title {
  font-family: {{ settings.type_header_font_family }};
  font-size: 28px;
  color: {{ settings.colors_text }};
}

.add-to-cart-button {
  background: {{ settings.colors_accent_1 }};
  color: white;
  padding: 15px 30px;
  border-radius: 25px;
  border: none;
}
```

5. Default Styles (`snippets/default_styles.liquid`):
```liquid
.featured-product-block {
  padding: 15px;
  background: white;
  border: 1px solid #eee;
}

.product-title {
  font-family: system-ui, sans-serif;
  font-size: 20px;
  color: #333;
}

.add-to-cart-button {
  background: #000;
  color: white;
  padding: 10px 20px;
  border-radius: 4px;
  border: none;
}
```

6. Common Styles (`assets/block.css`):
```css
/* Shared styles that apply regardless of theme */
.featured-product-block {
  margin-bottom: 20px;
  box-sizing: border-box;
}

.product-price {
  margin: 10px 0;
}

.add-to-cart-button {
  cursor: pointer;
  transition: opacity 0.2s;
}

.add-to-cart-button:hover {
  opacity: 0.9;
}
```

This structure allows you to:
- Keep theme-specific styles separated and organized
- Easily add support for new themes
- Maintain default styles for unsupported themes
- Share common styles across all themes
- Use theme settings variables where available

You can add more theme-specific files as needed, and the style switcher will automatically use the appropriate styles based on the active theme.
