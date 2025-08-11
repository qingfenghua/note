vmess://eyJ2IjoyLCJwcyI6IjIzM2JveS10Y3AtNzIuMTguODIuODEiLCJhZGQiOiI3Mi4xOC44Mi44MSIsInBvcnQiOiIzMjY1IiwiaWQiOiI4YjVmNDVmYy03NDBiLTQ5YmYtOTEzMi1lNzZhZWFlMzM1MzMiLCJhaWQiOiIwIiwibmV0IjoidGNwIiwidHlwZSI6Im5vbmUiLCJwYXRoIjoiIn0=


process.on('uncaughtException', (error) => {
  console.error('未捕获的异常:', error);
  process.exit(1);
});
