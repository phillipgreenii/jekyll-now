---
layout: post
title: Primefaces 3.0 fileUpload component on GAE
date: '2012-04-10T09:54:00.000-04:00'
author: Phillip Green II
tags:
- org.primefaces.webapp.filter.FileUploadFilter
- java
- jsr-314
- primefaces
- google app engine
- FileUploadFilter
- jsf
- gae
modified_time: '2012-04-10T09:54:00.248-04:00'
blogger_id: tag:blogger.com,1999:blog-3096944600800047027.post-7766739424383127420
blogger_orig_url: http://coder-in-training.blogspot.com/2012/04/primefaces-30-fileupload-component-on.html
---
I was recently working on a project where I used JSF 2.0 and Primefaces 3.0 for an application deployed to Google App Engine (GAE).  The particular problem was dealing with the fileUpload component.  In the [documentation][primefaces-docs] for Primefaces, it describes the need to configure `org.primefaces.webapp.filter.FileUploadFilter` so that the multipart request will be handled correctly. Normally, JSF ignores them.  The problem with `FileUploadFilter` is that it tries to write a temporary file which is not allowed on GAE.  I got around this limit by creating my own filter based on `FileUploadFilter`.

My initial solution was to save the temporary files as blobs,
but keeping the files in memory was much simpler.  I used `FileUploadFilter` as a reference and built an in-memory only `org.apache.commons.fileupload.FileItem` implementation.

**Warning: the whole file will be in memory, so this could cause resource issues.  For my particular use, I only was dealing with small files (<1MB) being uploaded infrequently.**

####ii.green.phillip.primefaces.GaeFileUploadFilter
```java
package ii.green.phillip.primefaces;

import java.io.IOException;
import java.util.logging.Level;
import java.util.logging.Logger;
import javax.servlet.*;
import javax.servlet.http.HttpServletRequest;
import org.apache.commons.fileupload.FileItemFactory;
import org.apache.commons.fileupload.servlet.ServletFileUpload;
import org.primefaces.webapp.MultipartRequest;
import org.primefaces.webapp.filter.FileUploadFilter;

/**
* Works like FileUploadFilter, but store files not as files because of restrictions from Google App Engine.
*
* @see FileUploadFilter
* @author pdgreen
*/
public class GaeFileUploadFilter implements Filter {

  private final static Logger logger = Logger.getLogger(GaeFileUploadFilter.class.getName());

  public void init(FilterConfig filterConfig) throws ServletException {
    if (logger.isLoggable(Level.FINE)) {
      logger.fine("FileUploadFilter initiated successfully");
    }
  }

  public void doFilter(ServletRequest request, ServletResponse response, FilterChain filterChain) throws IOException, ServletException {
    HttpServletRequest httpServletRequest = (HttpServletRequest) request;
    boolean isMultipart = ServletFileUpload.isMultipartContent(httpServletRequest);

    if (isMultipart) {
      if (logger.isLoggable(Level.FINE)) {
        logger.fine("Parsing file upload request");
      }

      final FileItemFactory fileItemFactory = new InMemoryFileItemFactory();

      ServletFileUpload servletFileUpload = new ServletFileUpload(fileItemFactory);
      MultipartRequest multipartRequest = new MultipartRequest(httpServletRequest, servletFileUpload);

      if (logger.isLoggable(Level.FINE)) {
        logger.fine("File upload request parsed succesfully, continuing with filter chain with a wrapped multipart request");
      }
      try {
        filterChain.doFilter(multipartRequest, response);
      } catch (ServletException ex) {
        logger.log(Level.SEVERE, "servlet exception occured", ex);
        throw ex;
        } catch (IOException ex) {
          logger.log(Level.SEVERE, "io exception occured", ex);
          throw ex;
        }
        } else {
          filterChain.doFilter(request, response);
        }
      }

      public void destroy() {
        if (logger.isLoggable(Level.FINE)) {
          logger.fine("Destroying FileUploadFilter");
        }
      }
    }
```

####ii.green.phillip.primefaces.InMemoryFileItemFactory
```java
package ii.green.phillip.primefaces;

import org.apache.commons.fileupload.FileItem;
import org.apache.commons.fileupload.FileItemFactory;

/**
* Creates {@link InMemoryFileItem}s.
* @author pdgreen
*/
public class InMemoryFileItemFactory implements FileItemFactory {

  @Override
  public FileItem createItem(String fieldName, String contentType, boolean isFormField, String fileName) {
    return new InMemoryFileItem(fieldName, contentType, isFormField, fileName);
  }
}
```

####ii.green.phillip.primefaces.InMemoryFileItem
```java
package ii.green.phillip.primefaces;

import java.io.*;
import org.apache.commons.fileupload.*;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

/**
* Only stores the file in memory.
*
* @author pdgreen
*/
public class InMemoryFileItem implements FileItem {

  private final static Logger LOGGER = LoggerFactory.getLogger(InMemoryFileItem.class);
  public static final String DEFAULT_CHARSET = "ISO-8859-1";
  private String fieldName;
  private String contentType;
  private boolean isFormField;
  private String fileName;
  private byte[] content = new byte[0];

  /**
  * Constructs a new
  * <code>GaeFileItem</code> instance.
  *
  * @param fieldName The name of the form field.
  * @param contentType The content type passed by the browser or
  * <code>null</code> if not specified.
  * @param isFormField Whether or not this item is a plain form field, as opposed to a file upload.
  * @param fileName The original filename in the user's filesystem, or
  * <code>null</code> if not specified.
  */
  public InMemoryFileItem(String fieldName,
  String contentType, boolean isFormField, String fileName) {
    this.fieldName = fieldName;
    this.contentType = contentType;
    this.isFormField = isFormField;
    this.fileName = fileName;
  }

  @Override
  public InputStream getInputStream()
  throws IOException {
    return new ByteArrayInputStream(content);
  }

  @Override
  public String getContentType() {
    return contentType;
  }

  @Override
  public String getName() {
    return fileName;
  }

  @Override
  public boolean isInMemory() {
    return true;
  }

  @Override
  public long getSize() {
    return content.length;
  }

  @Override
  public byte[] get() {
    return content;
  }

  @Override
  public String getString(final String charset)
  throws UnsupportedEncodingException {
    return new String(get(), charset);
  }

  @Override
  public String getString() {
    try {
      return new String(content, DEFAULT_CHARSET);
      } catch (UnsupportedEncodingException e) {
        return new String(content);
      }
    }

    @Override
    public void write(File file) throws Exception {
      FileOutputStream fout = null;
      try {
        fout = new FileOutputStream(file);
        fout.write(get());
        } finally {
          if (fout != null) {
            fout.close();
          }
        }
      }

      @Override
      public void delete() {
        content = new byte[0];
      }

      @Override
      public String getFieldName() {
        return fieldName;
      }

      @Override
      public void setFieldName(String fieldName) {
        this.fieldName = fieldName;
      }

      @Override
      public boolean isFormField() {
        return isFormField;
      }

      @Override
      public void setFormField(boolean state) {
        isFormField = state;
      }

      @Override
      public OutputStream getOutputStream()
      throws IOException {

        return new ByteArrayOutputStream() {

          @Override
          public synchronized void reset() {
            super.reset();
            content = toByteArray();
          }

          @Override
          public void flush() throws IOException {
            super.flush();
            content = toByteArray();
          }

          @Override
          public void close() throws IOException {
            super.close();
            content = toByteArray();
          }
        };
      }
    }
```

## Conclusion
The above code worked great for my requirements.  
If my requirements were to expand, I would try storing the temporary files as blobs.


## References
 * [Primefaces Documentation][primefaces-docs]

[primefaces-docs]: <http://primefaces.org/documentation.html> "Primefaces Documentation"
