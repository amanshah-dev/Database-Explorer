-- Create function to generate algorithm sign using HMAC
create function algorithm_sign(signables text, secret text, algorithm text) returns text
    immutable
    language sql
as
$$
WITH
  alg AS (
    SELECT CASE
      WHEN algorithm = 'HS256' THEN 'sha256'
      WHEN algorithm = 'HS384' THEN 'sha384'
      WHEN algorithm = 'HS512' THEN 'sha512'
      ELSE '' END AS id)  -- hmac throws error
SELECT url_encode(hmac(signables, secret, alg.id)) FROM alg;
$$;

-- Alter function owner
alter function algorithm_sign(text, text, text) owner to learner;
