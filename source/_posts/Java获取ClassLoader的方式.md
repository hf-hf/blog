---
title: Java获取ClassLoader的方式
tags:
  - Java
password:
  - hf123456
cover: /upload/homePage/20190412091707.jpg
abbrlink: 2038a319
categories: uncategorized
date: 2019-04-12 16:25:33
---
ClassLoader.getSystemClassLoader();
Thread.currentThread().getContextClassLoader();

classLoader.getResources(FILTER_PROPERTY_NAME);

```
private Properties loadFilterConfig() throws IOException {
		Properties filterProperties = new Properties();

    loadFilterConfig(filterProperties, ClassLoader.getSystemClassLoader());
    loadFilterConfig(filterProperties, Thread.currentThread().getContextClassLoader());

    return filterProperties;
}

private void loadFilterConfig(Properties filterProperties, ClassLoader classLoader) throws IOException {
    if (classLoader == null) {
        return;
    }

    for (Enumeration<URL> e = classLoader.getResources(FILTER_PROPERTY_NAME); e.hasMoreElements();) {
        URL url = e.nextElement();

        Properties property = new Properties();

        InputStream is = null;
        try {
            is = url.openStream();
            property.load(is);
        } finally {
            if (is != null) {
                is.close();
            }
        }

        filterProperties.putAll(property);
    }
}
```
